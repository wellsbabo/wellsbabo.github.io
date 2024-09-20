---
layout: post
title:  Creating Credentials for OAuth2 in Google Cloud Console - Understanding OAuth 2.0 Client ID vs. Service Account
categories: [Google API, OAuth2.0]
tags: [Google Cloud, OAuth2]
description: OAuth 2.0 Client ID and Service Account authentication in Google Cloud Console.
---

When setting up credentials in Google Cloud Console to use OAuth2, you’ll come across two options: **OAuth 2.0 Client ID** and **Service Account**. 

While working on the Google Play Store receipt validation and processing API, I delved deeper into the differences between these two options and wanted to summarize them here.

To put it simply, when using an **OAuth 2.0 Client ID**, you need to register a **Redirect URL**, but with a **Service Account**, no Redirect URL is required.

## Basic Concepts of OAuth 2.0 and Service Account Authentication 🙄

### OAuth 2.0 Client ID
OAuth 2.0 is an authorization protocol that allows third-party applications to access user data in a limited way. Its main features include:

1. **User-Centric**: The end user grants access to their data.
2. **Delegated Access**: It allows access to specific resources without sharing the user’s password.
3. **Token-Based**: Access tokens are used to manage authentication and authorization.
4. **Requires Redirect**: A redirect URL is needed to obtain user consent.

### Service Account

A service account is a special type of account meant for applications or virtual machines rather than individual users. Its main features include:

1. **Application-Centric**: It operates on behalf of a specific service or application.
2. **Key-Based Authentication**: It uses a JSON key file for authentication.
3. **Direct Access**: It can directly access resources without user intervention.
4. **No Redirect Needed**: Since there’s no user consent process, no redirect URL is necessary.

The key difference between the two methods lies in **whether user intervention is required and the flow of authentication**. OAuth 2.0 requires user consent, whereas the service account operates automatically in the background.

## OAuth 2.0 Client Authentication 🙄

### How It Works (Google Example)

1. A user attempts to log in to an application (client) using their Google account.
2. The application redirects the user to Google’s OAuth 2.0 authentication page on the Google authentication server (Server A).
3. The user logs in with their Google account on the authentication server (Server A).
4. After logging in, the user is prompted to grant the application the requested permissions.
5. Once the user consents, the Google authentication server (Server A) redirects the user to the pre-registered redirect URL, passing along an authorization code.
6. The application uses this authorization code to request an access token from Google’s authentication server (Server A).
7. After the access token is issued, the application uses it to access user data from Google’s resource server (Server B).

In this flow, Google’s authentication server (Server A) handles authentication and authorization, while the resource server (Server B) stores and provides actual user data (e.g., profile information, email, etc.).

### Redirect URL

The reason a redirect URL is required in this process is for security.

Google’s authentication server will only send the authorization code to the registered redirect URL. If this registration process didn’t exist, a malicious third party could potentially intercept the authorization code.

Furthermore, without the redirect URL, the user wouldn’t be able to return to the application after completing authentication, making it impossible to complete the process.

## Service Account Authentication 🙄

### How It Works

1. The application server uses the JSON key file, issued by Google Cloud Console, to combine with the desired scope and create a JWT token.
2. The application server sends the generated JWT token to Google’s authentication server.
3. Google’s authentication server validates the JWT token and, if valid, issues an access token.
4. The application server uses this access token to request access to user data from Google’s resource server.

### Role of the JSON Key File

The JSON key file contains the service account’s authentication information. This file is issued by Google Cloud Console and stored on the application server.

The application server combines the key file with the desired scope to generate a JWT token, which is then sent to Google’s authentication server for validation.

### Why No Redirect URL is Needed

Since the service account generates the token internally and simply needs validation from the authentication server, a redirect URL is unnecessary.

## Key Differences Between the Two Methods 🙄

1. **User Intervention**: OAuth 2.0 requires user intervention, whereas service accounts do not.
2. **Authentication Flow**: OAuth 2.0 requires a redirect URL for user consent, while service accounts do not.
3. **Security**: OAuth 2.0 maintains security through the redirect URL, while service accounts rely on key files for security.
4. **Purpose**: OAuth 2.0 is designed to provide limited access to user data, whereas service accounts are meant to operate on behalf of applications or services.

### Comparison of Implementation Complexity

OAuth 2.0 client authentication involves a more complex flow. It requires registering a redirect URL to obtain user consent, receiving an authorization code, and then using that code to request an access token.

Service account authentication, on the other hand, is relatively simpler, using a key file to authenticate.

### Difference in Redirect URL Usage

With service account authentication, the application server creates a token directly using the JSON key file, sends it to the server, and receives an access token after verification. **(Authentication is completed in a single HTTP request-response cycle)**.

However, with OAuth 2.0 client authentication, the user must log in through a browser, grant permissions, and return to the application with the authorization code (requiring the redirect URL), which is then exchanged for an access token. **(Multiple redirections are involved)**.

### Pros and Cons of Each Method

#### Service Account Authentication
- **Pros**:
    - Automated authentication without user intervention.
    - Suitable for server-to-server communication.
    - Simpler to implement.
- **Cons**:
    - Requires secure management of the sensitive JSON key file.
    - Not suitable for operations in a user context.

#### OAuth 2.0 Client Authentication
- **Pros**:
    - Secure, user-consent-based authentication process.
    - Suitable for operations requiring user context.
    - Easier to manage token renewal and permissions.
- **Cons**:
    - More complex authentication flow and implementation.
    - Requires user intervention, limiting automation.

### Appropriate Use Cases

- **Service Account Authentication**:
    - When server-to-server communication is needed.
    - For running background tasks or batch processes.

- **OAuth 2.0 Client Authentication**:
    - In web or mobile applications that need access to user data.
    - When different permissions are required for each user.
    - When limited access needs to be granted to third-party applications.
