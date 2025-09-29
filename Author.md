# Đang nhập và xác thực

## Đang nhập

Request

```jsx
// query
query CheckUserExists($email: String!, $captchaToken: String) {
     checkUserExists(email: $email, captchaToken: $captchaToken) {
        exists
        availableWorkspacesCount
        isEmailVerified
        __typename
    }
}
// VARIABLES
{"email": "tim@apple.dev"} 
```

Response

```jsx

// Email tồn tại
{
    "data": {
        "checkUserExists": {
            "__typename": "CheckUserExistOutput",
            "exists": true,
            "availableWorkspacesCount": 2,
            "isEmailVerified": true
        }
    }
}
// Email không tồn tại
{
    "data": {
        "checkUserExists": {
            "__typename": "CheckUserExistOutput",
            "exists": false,
            "availableWorkspacesCount": 0,
            "isEmailVerified": false
        }
    }
}
```

## B2: Check Email và mật khẩu

Request

```jsx
// QUERY 
mutation GetLoginTokenFromCredentials(
  $email: String!, 
  $password: String!, 
  $captchaToken: String, 
  $origin: String!
) {
  getLoginTokenFromCredentials(
    email: $email
    password: $password
    captchaToken: $captchaToken
    origin: $origin
  ) {
    loginToken {
      ...AuthTokenFragment
      __typename
    }
    __typename
  }
}

fragment AuthTokenFragment on AuthToken {
  token
  expiresAt
  __typename
}
// VARIABLES
{
"email": "tim@apple.dev",
"password": "tim@apple.dev", 
"origin": "http://localhost:3001"
}
```

Response

```jsx
// Tài khoản và mật khẩu chính xác
{
    "data": {
        "getLoginTokenFromCredentials": {
            "__typename": "LoginToken",
            "loginToken": {
                "__typename": "AuthToken",
                "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiTE9HSU4iLCJzdWIiOiJ0aW1AYXBwbGUuZGV2Iiwid29ya3NwYWNlSWQiOiIzYjhlNjQ1OC01ZmMxLTRlNjMtODU2My0wMDhjY2RkYWE2ZGIiLCJhdXRoUHJvdmlkZXIiOiJwYXNzd29yZCIsImlhdCI6MTc1ODI1NDE0MiwiZXhwIjoxNzU4MjU1MDQyfQ.bQyacyRA19o8sjzpsNxrBCWIQHyyu1LVnxIhQVqLo9k",
                "expiresAt": "2025-09-19T04:10:42.009Z"
            }
        }
    }
}
// Sai mật khẩu
{
    "errors": [
        {
            "message": "Wrong password",
            "extensions": {
                "subCode": "FORBIDDEN_EXCEPTION",
                "userFriendlyMessage": "Wrong password",
                "code": "FORBIDDEN"
            }
        }
    ],
    "data": null
}
```

## 2FA với Google Authentication

- access token : Token dùng để xác thực người dùng đã xác thực 2FA
- loginToken: Token dùng để xác thực người dùng chưa xác thực 2FA

(Được sinh ra khi đang nhập bằng email, password )

- origin: Url website

### Tích hợp 2FA

- Sau khi nhập tài khoản và mật khẩu trên thiết bị lần đầu tiên sẽ phải tạo mã xác thực trên google authentication
- Từ các lần sau bỏ qua bước này.

Request

```jsx
// QUERY
mutation initiateOTPProvisioningForAuthenticatedUser {
  initiateOTPProvisioningForAuthenticatedUser {
    uri
  }
}
// VARIABLES
{
  "loginToken": "",
  "origin": "http://localhost:3001", 
}
```

Response

- Sử dụng uri trả về tạo mã QR và yêu cầu người dùng sử dụng google authentication quét mã QR

```jsx
// Lỗi Login token hết hạn
{
    "errors": [
        {
            "message": "Token has expired.",
            "extensions": {
                "userFriendlyMessage": "You must be authenticated to perform this action.",
                "subCode": "UNAUTHENTICATED",
                "code": "UNAUTHENTICATED"
            }
        }
    ],
    "data": null
}
// Lỗi authentication đã được cài dặt trước đó
{
    "errors": [
        {
            "message": "A two factor authentication method has already been set. Please delete it and try again.",
            "extensions": {
                "code": "INTERNAL_SERVER_ERROR",
                "userFriendlyMessage": "An error occurred."
            }
        }
    ],
    "data": null
}
// Thành công trả về loginToken
{
    "data": {
        "initiateOTPProvisioning": {
            "uri": "otpauth://totp/Twenty%20-%20YCombinator:phil.schiler%40apple.dev?secret=AUND6UTCCVNXMWYY&period=30&digits=6&algorithm=SHA1&issuer=Twenty%20-%20YCombinator"
        }
    }
}
```

### Xác thực 2FA

Request

```graphql
// QUERY
mutation getAuthTokensFromOTP($otp: String!, $loginToken: String!, $origin: String!) {
  getAuthTokensFromOTP(
    otp: $otp
    loginToken: $loginToken
    origin: $origin
  ) {
    tokens {
      accessToken {
        token
        expiresAt
      },
      refreshToken{
            token
        expiresAt
      }
    }
  }
}
// VARIABLES
{
  "otp": "063257", // Mã otp trong ứng dụng google authentication
  "loginToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiTE9HSU4iLCJzdWIiOiJqb255Lml2ZUBhcHBsZS5kZXYiLCJ3b3Jrc3BhY2VJZCI6IjNiOGU2NDU4LTVmYzEtNGU2My04NTYzLTAwOGNjZGRhYTZkYiIsImF1dGhQcm92aWRlciI6InBhc3N3b3JkIiwiaWF0IjoxNzU4MjY0NDg3LCJleHAiOjE3NTgyNjUzODd9._cUu-e0sEiKUcHXqGu4DSKOCGc-INEf7-hXBAY6TGU4",
  "origin": "http://localhost:3001"
}
```

Response

```graphql
// Lỗi sai OTP
{
    "errors": [
        {
            "message": "Invalid OTP",
            "extensions": {
                "subCode": "INVALID_OTP",
                "userFriendlyMessage": "Invalid verification code. Please try again.",
                "code": "BAD_USER_INPUT"
            }
        }
    ],
    "data": null
}
// Lỗi tokenLogin không đúng
{
    "errors": [
        {
            "message": "Token invalid.",
            "extensions": {
                "userFriendlyMessage": "You must be authenticated to perform this action.",
                "subCode": "UNAUTHENTICATED",
                "code": "UNAUTHENTICATED"
            }
        }
    ],
    "data": null
}
// Thành công trả về accessToken và refreshToken
{
    "data": {
        "getAuthTokensFromOTP": {
            "tokens": {
                "accessToken": {
                    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIyMDIwMjAyMC0zOTU3LTQ5MDgtOWMzNi0yOTI5YTIzZjgzNTciLCJ1c2VySWQiOiIyMDIwMjAyMC0zOTU3LTQ5MDgtOWMzNi0yOTI5YTIzZjgzNTciLCJ3b3Jrc3BhY2VJZCI6IjNiOGU2NDU4LTVmYzEtNGU2My04NTYzLTAwOGNjZGRhYTZkYiIsIndvcmtzcGFjZU1lbWJlcklkIjoiMjAyMDIwMjAtNzdkNS00Y2I2LWI2MGEtZjRhODM1YTg1ZDYxIiwidXNlcldvcmtzcGFjZUlkIjoiMjAyMDIwMjAtZTEwYS00YzI3LWE5MGItYjA4YzU3YjAyZDQ1IiwidHlwZSI6IkFDQ0VTUyIsImF1dGhQcm92aWRlciI6InBhc3N3b3JkIiwiaWF0IjoxNzU4MjY2ODY3LCJleHAiOjE3NTgyNjg2Njd9.R9VYZaLCYyBK2n5cR1JtKWMFX3lWYEIRw8QEy4B5qLU",
                    "expiresAt": "2025-09-19T07:57:47.204Z"
                },
                "refreshToken": {
                    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIyMDIwMjAyMC0zOTU3LTQ5MDgtOWMzNi0yOTI5YTIzZjgzNTciLCJ3b3Jrc3BhY2VJZCI6IjNiOGU2NDU4LTVmYzEtNGU2My04NTYzLTAwOGNjZGRhYTZkYiIsImF1dGhQcm92aWRlciI6InBhc3N3b3JkIiwidGFyZ2V0ZWRUb2tlblR5cGUiOiJBQ0NFU1MiLCJzdWIiOiIyMDIwMjAyMC0zOTU3LTQ5MDgtOWMzNi0yOTI5YTIzZjgzNTciLCJ0eXBlIjoiUkVGUkVTSCIsImlhdCI6MTc1ODI2Njg2NywiZXhwIjoxNzYzNDUwODY3LCJqdGkiOiI4NDE4YzI1OC1kMjUyLTQ4Y2ItODY2YS02MjljZTYwZTA5OGYifQ.-l4JMNWA3Y0YWvySZDhM3VGF7P7AdlFUi8cADhjx5WE",
                    "expiresAt": "2025-11-18T07:27:47.218Z"
                }
            }
        }
    }
}
```

### Verified Email

Request

```graphql
// Query
mutation resendEmailVerificationToken($email: String!, $origin: String!) {
  resendEmailVerificationToken(
     email: $email
    origin: $origin
  ) {
    success
  }
}

// Data
{
  "email": "tinhnq@phanmemmkt.vn",
  "origin": "http://localhost:3001"
}
```

Response

```json
// Email đã xác thực trước đó
{
    "errors": [
        {
            "message": "Email already verified",
            "extensions": {
                "subCode": "EMAIL_ALREADY_VERIFIED",
                "userFriendlyMessage": "Email already verified.",
                "code": "BAD_USER_INPUT"
            }
        }
    ],
    "data": null
}

// Gửi email xác thực thành công, chờ người dùng xác thực
{
    "data": {
        "resendEmailVerificationToken": {
            "success": true
        }
    }
}
```
