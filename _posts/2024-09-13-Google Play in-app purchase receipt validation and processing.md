---
layout: post
title:  Google Play in-app purchase receipt validation and processing
categories: [Google API, Backend Development, spring-boot]
tags: [In-App Purchase, Google Play, Google Cloud, OAuth2]
description: Guide to validating and processing Google Play in-app purchase receipts using Spring Boot and Google Cloud API, with steps on setting up credentials, handling receipt verification, and managing errors.
---

### Current Requirements
- Google Play In-App Billing (One-time product for ad removal, not consumable items)

### Processing Flow
![processing flow](/assets/img/posts/240916/20240913_inApp_remove_ads_process.png)
1. When the server starts, obtain an Access Token to access the Google Play API through the Google Cloud API (automatically renewed when expired)
2. Receive receipt information from the client when a purchase is made
3. Use the receipt information to access the Google Play Console API and verify if it's a valid purchase receipt
4. If confirmed as a valid purchase receipt, access the Google Play Console API again to process the receipt
5. After processing the receipt, add the user to the ad-free users table and complete the purchase process

There are a few things to know to proceed with this process.

First, a part that was a bit confusing: **To access the Google Play Console API, you need credentials issued by Google.**
However, this should be obtained through the Google Cloud Console, not the Google Play Console.

Secondly, to access the Google Play API using the credentials obtained from the Google Cloud Console, **the Google Cloud account and Google Play account must be linked**.

Let's start by acknowledging these two points.

---

Before writing code, there's a part that needs to be done first.
Since it's related to settings, not code, I'll briefly mention the important points.
You can find a lot of related information by searching.

#### 1. Google Cloud Console Setup
Create an account in the Google Cloud Console and generate 'Credentials' in the 'APIs & Services' tab. At this point, create a **'Service Account'**, not an 'OAuth 2.0 Client ID'.

Although 'OAuth 2.0 Client ID' can be used, what I'm trying to do now is receipt authentication processing through server-to-server communication, so we use 'Service Account' specialized for server-to-server service provision, not 'OAuth 2.0 Client ID' specialized for login.

After creating the service account, you can receive the account information as JSON. Let's download it for later use. The name will probably be in the form of project_id-xxxxxxxxx.json, but for convenience, I'll call it profile.json here.

Then change the role of the created service account to 'Owner' or 'Editor', or if you want to be more specific, 'Service Account Token Creator' is sufficient for receipt processing.

Then change the Google Play Android Developer API to 'Enabled' in the 'API Library' tab.

#### 2. Google Play Console Setup
Link with the Google Cloud service account and register the app. Then register temporary items you want to test for purchase.

In the above process, you need to register the redirect URL to your current Spring server address in the Google Cloud Console. (You don't need to do this in the Google Play Console)

---

Now let's start writing the actual code.

In fact, since we're using the Google API, if you've gone through the above setup process well, writing the code itself is not difficult. The slightly tricky part was inputting the appropriate parameters. (I'll explain this after explaining the code.)

I created a separate class called InAppPurchaseService.java within the project to write in-app purchase related code, taking responsibility for all communication with the Google API here, and implemented the purchase processing part in the existing main service, Service.java.

![in_server flow](/assets/img/posts/240916/20240913_inApp_purchase_in_server_flow.png)

#### InAppPurchaseService.java
This code is the core code for in-app purchase processing.

```java
/**
 * Service class for handling in-app purchases using Google Play Billing Library.
 * This class provides methods for verifying and acknowledging purchase receipts.
 */
public class InAppPurchaseService {

  private final AndroidPublisher androidPublisher;

  /**
   * Constructor that initializes the AndroidPublisher client.
   * It sets up the necessary credentials and builds the AndroidPublisher instance.
   *
   * @throws GeneralSecurityException if there's a security-related exception
   * @throws IOException if there's an I/O error when reading the credentials file
   */
  public InAppPurchaseService() throws GeneralSecurityException, IOException {
    String serviceAccountKeyFilePath =
        getClass().getClassLoader().getResource("profile.json").getPath();

    try {
      GoogleCredentials credentials =
          GoogleCredentials.fromStream(new FileInputStream(serviceAccountKeyFilePath))
              .createScoped(
                  Collections.singleton("https://www.googleapis.com/auth/androidpublisher"));
      log.info("Credentials successfully created: " + credentials);
      HttpRequestInitializer requestInitializer = new HttpCredentialsAdapter(credentials);	// Required for automatic token refresh

      // Create AndroidPublisher instance using the credentials
      this.androidPublisher =
          new AndroidPublisher.Builder(
                  GoogleNetHttpTransport.newTrustedTransport(),
                  GsonFactory.getDefaultInstance(),
                  requestInitializer)
              .setApplicationName("MyApplication")	//It must match the project name in Google Cloud Console
              .build();
      
      credentials.refresh();
      log.info("Access token: " + credentials.getAccessToken().getTokenValue());
    } catch (IOException e) {
      log.info("Error occurred while creating credentials: " + e.getMessage());
      e.printStackTrace();
      throw e;
    }
  }

  /**
   * Verifies the purchase receipt with Google Play.
   *
   * @param packageName The package name of the app
   * @param productId The ID of the product that was purchased
   * @param purchaseToken The token that was provided to the user after the purchase (receipt)
   * @return ProductPurchase object containing the purchase details
   * @throws ReceiptVerificationException if there's an error during verification
   */
  public ProductPurchase verifyReceipt(String packageName, String productId, String receipt)
      throws ReceiptVerificationException {
        try{
          return androidPublisher
          .purchases()
          .products()
          .get(packageName, productId, receipt)
          .execute();
        } catch (IOException e) {
          if (e instanceof GoogleJsonResponseException) {
                GoogleJsonResponseException gjre = (GoogleJsonResponseException) e;
                if (gjre.getStatusCode() == 401) {
                    throw new ReceiptVerificationException("Authentication error: Please check your API key or permissions.", e);
                } else if (gjre.getStatusCode() == 404) {
                    throw new ReceiptVerificationException("Receipt not found: Please check the package name, product ID, and purchase token.", e);
                }
            }
            throw new ReceiptVerificationException("Error occurred during receipt verification", e);
        }
  }

  /**
   * Acknowledges the purchase receipt with Google Play.
   *
   * @param packageName The package name of the app
   * @param productId The ID of the product that was purchased
   * @param receipt The token that was provided to the user after the purchase
   * @param userId The ID of the user who made the purchase
   * @throws ReceiptAcknowledgementException if there's an error during acknowledgement
   */
  public void acknowledgeReceipt(String packageName, String productId, String receipt, String userId) throws ReceiptAcknowledgementException {
    try {
      ProductPurchasesAcknowledgeRequest request = new ProductPurchasesAcknowledgeRequest()
      .setDeveloperPayload(userId);
        androidPublisher.purchases().products()
            .acknowledge(packageName, productId, receipt, request)
            .execute();
    } catch (IOException e) {
        if (e instanceof GoogleJsonResponseException) {
            GoogleJsonResponseException gjre = (GoogleJsonResponseException) e;
            if (gjre.getStatusCode() == 401) {
                throw new ReceiptAcknowledgementException("Authentication error: Please check your API key or permissions.", e);
            } else if (gjre.getStatusCode() == 404) {
                throw new ReceiptAcknowledgementException("Receipt not found: Please check the package name, product ID, and purchase token.", e);
            }
        }
        throw new ReceiptAcknowledgementException("Error occurred during receipt acknowledgement", e);
    }
  }

  /**
   * Custom exception class for receipt verification errors.
   */
  public static class ReceiptVerificationException extends IOException {
    public ReceiptVerificationException(String message, Throwable cause) {
        super(message, cause);
    }
  }

  /**
   * Custom exception class for purchase processing errors.
   */
  public static class PurchaseProcessingException extends Exception {
    public PurchaseProcessingException(String message) {
        super(message);
    }

    public PurchaseProcessingException(String message, Throwable cause) {
        super(message, cause);
    }
  }

  /**
   * Custom exception class for receipt acknowledgement errors.
   */
  public static class ReceiptAcknowledgementException extends Exception {
    public ReceiptAcknowledgementException(String message) {
        super(message);
    }

    public ReceiptAcknowledgementException(String message, Throwable cause) {
        super(message, cause);
    }
  }
}
```


Here, the `verifyReceipt` method receives `ProductPurchase` representing the in-app product purchase status as a return value, which looks like this:

```json
{
  "kind": string,
  "purchaseTimeMillis": string,
  "purchaseState": integer,
  "consumptionState": integer,
  "developerPayload": string,
  "orderId": string,
  "purchaseType": integer,
  "acknowledgementState": integer,
  "purchaseToken": string,
  "productId": string,
  "quantity": integer,
  "obfuscatedExternalAccountId": string,
  "obfuscatedExternalProfileId": string,
  "regionCode": string,
  "refundableQuantity": integer
}

```

For more details, refer to the link:

https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.products?hl=en#ProductPurchase

Here, `purchaseState` distinguishes the current purchase status, where 0 indicates that the purchase is completed.

On the other hand, the `acknowledgeReceipt` method has no return value because the response body is empty when the acknowledge is processed normally.

However, it's sufficient to handle exceptions for the response code to check if it was processed normally.

---

Now let's check the part that calls and uses the methods of `InAppPurchaseService.java` we looked at above and wrap up.

Since the point of this article is to verify and process receipts, I'll omit the purchase process after that. Probably because each application has different purchase processing logic, it won't be very meaningful.

Anyway, the part that calls the receipt processing method we created above is as follows:

```java
/**
 * Process in-app purchase and verify receipt
 *
 * @param inAppPurchaseDTO DTO containing purchase information
 * @return Result of purchase processing
 * @throws GeneralSecurityException If there's a security-related exception
 * @throws IOException If there's an I/O error
 */
@Transactional(rollbackFor = Exception.class)
public int inAppPurchase(InAppPurchaseDTO inAppPurchaseDTO)
    throws GeneralSecurityException, IOException {
      try {
        // Verify the purchase receipt
        ProductPurchase productPurchase = inAppPurchaseService.verifyReceipt(
            inAppPurchaseDTO.getPackageName(),
            inAppPurchaseDTO.getProductId(),
            inAppPurchaseDTO.getReceipt()
        );

        int purchaseProcessResult = processPurchase(productPurchase, inAppPurchaseDTO);

        return purchaseProcessResult;
    } catch (ReceiptVerificationException e) {
      log.error("Error occurred during receipt verification: " + e.getMessage());
      throw new RuntimeException("Receipt verification failed", e);
    } catch (InAppPurchaseService.PurchaseProcessingException e) {
      log.error("Error occurred during purchase processing: " + e.getMessage());
      throw new RuntimeException("Purchase processing failed", e);
    } catch (Exception e) {
      log.error("Unexpected error occurred: " + e.getMessage());
      throw new RuntimeException("Error during in-app purchase", e);
    }
}

/**
 * Process the purchase after receipt verification
 * 
 * @param productPurchase Verified purchase information
 * @param inAppPurchaseDTO DTO containing purchase details
 * @return Result of purchase processing
 * @throws PurchaseProcessingException If there's an error during purchase processing
 */
private int processPurchase(ProductPurchase productPurchase, InAppPurchaseDTO inAppPurchaseDTO) throws PurchaseProcessingException {
  // 1. Check purchase state
  if(productPurchase.getPurchaseState() != 0){
    throw new PurchaseProcessingException("Purchase not completed. State: " + productPurchase.getPurchaseState());
  }
  log.info("Purchase completed. State: " + productPurchase.getPurchaseState());

  // 2. Prevent duplicate processing
  if(isPurchaseAlreadyProcessed(productPurchase.getOrderId())){
    throw new PurchaseProcessingException("This purchase has already been processed.");
  }
  log.info("Duplicate processing prevention completed");

  try {
    // 3. Update user status (e.g., remove ads)
    // Implementation details omitted

    // 4. Acknowledge the receipt
    inAppPurchaseService.acknowledgeReceipt(
      inAppPurchaseDTO.getPackageName(),
      inAppPurchaseDTO.getProductId(),
      inAppPurchaseDTO.getReceipt(),
      inAppPurchaseDTO.getPlayerId()
    );
    log.info("Receipt acknowledgement completed");

    // 5. Save receipt record
    // Implementation details omitted

    return 1;
  } catch(InAppPurchaseService.ReceiptAcknowledgementException e){
    log.error("Error occurred during receipt acknowledgement: " + e.getMessage());
    throw new RuntimeException("Receipt acknowledgement failed", e);
  } catch(Exception e){
    log.error("Unexpected error occurred: " + e.getMessage());
    throw new RuntimeException("Error during in-app purchase processing", e);
  }
}

/**
 * Check if a purchase has already been processed
 * 
 * @param orderId The order ID to check
 * @return true if the purchase has already been processed, false otherwise
 */
private boolean isPurchaseAlreadyProcessed(String orderId) {
  return mapper.isPurchaseAlreadyProcessed(orderId);
}
```

Here, you just need to call and check the response value and continue the process, so there's no big problem.

However, it's important to clarify what information should be used from the `InAppPurchaseDTO` used in the above code to request the Google API.

```java
public class InAppPurchaseDTO {
  private String playerId;	// unique value that distinguishes the user
  private String packageName;	// app package name. ex) com.test.app
  private String transactionId;	// order id. ex) GPA.XXXX.XXXXX.XXXXX.XXX
  private String productId;	// purchase product ID. ex) ad_remover
  private String receipt;	// token. ex) dipjcnhompnmpbkokmjjdfam.AO-XXXwKJo931-qI0crOddpS1hKm2-vOfXGS2cxJfRsiEepNF4lfjqYTU6deUs6_hAhofJdBD9Az_CbfKm6xQJ5hAhofJdVNXH7zGzxfWt0gd2y3_pehztA
}
```

I searched for several related articles, but the explanations were ambiguous, so I kept sending packageName, productId, transactionId. It seems that it was correct to send transactionId before, but it shouldn't be sent that way now. I could identify the problem by reading the following post:

https://stackoverflow.com/questions/46055214/google-play-developer-api-400-invalid-value-inapppurchases

```java
ProductPurchase productPurchase = inAppPurchaseService.verifyReceipt(
    inAppPurchaseDTO.getPackageName(),
    inAppPurchaseDTO.getProductId(),
    inAppPurchaseDTO.getReceipt()
);
```

You need to send packageName, productId, receipt(token) to the Google API in this way to receive a normal response.

---

So far, we've looked at the process of verifying and processing in-app purchase receipts through the Google API.

Now I'll summarize the parts where I got stuck in this process. (The reason I wrote this article)

### 1. Service Account Creation & Permission Granting and In-App Product Registration Order Issue (401 Error)

![401 Error](/assets/img/posts/240916/20240913_inApp_purchase_googleAPI_401.png)

I struggled quite a bit with this as well.

Everything was done properly, but a 401 error came out like below.

The conclusion is, as in the title, the timing of granting permissions to the service account should precede the timing of in-app product registration.

In simple terms, if you create a service account and grant permissions with products already registered, it's treated as not having permissions for already registered products, resulting in a 401 error.

If you're already in this situation, don't create the in-app product again. Instead, go to view in-app products, change the name or product description roughly, save the changes, and try again.

[[Links to related Stack Overflow and blog posts]](https://stackoverflow.com/questions/43536904/google-play-developer-api-the-current-user-has-insufficient-permissions-to-pe/60691844#60691844)

[https://sobob.tistory.com/51](https://sobob.tistory.com/51)


### 2. Google API Parameter Issue (400 Error)

![400 Error](/assets/img/posts/240916/20240913_inApp_purchase_googleAPI_400.png)

As mentioned earlier, due to version changes and various incorrect information, it's confusing which values should be used as parameters.

At this time, if you use incorrect parameters, you'll get a 400 Error, so check carefully.

[Link to related Stack Overflow post]

#### When such problems occur, it's always clear to look at the official documentation.
https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.products
