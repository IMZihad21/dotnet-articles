## Introduction

Two-factor authentication (2FA) adds an extra layer of security to your application by requiring users to provide a second form of verification during login. In this article, we'll walk through a practical implementation of time-based one-time password (TOTP) 2FA in ASP.NET Core, complete with QR code setup and OTP validation.

## Why 2FA Matters

Before diving into the code, let's understand why 2FA is crucial:

- Mitigates password-related risks
- Adds defense against phishing attacks
- Complies with modern security standards
- Protects sensitive user data

## The Implementation Blueprint

Our solution includes three core components:

1. Traditional email/password login with 2FA check
2. OTP verification endpoint
3. QR code generation for authenticator apps

### 1. Get Authenticator QR Code for 2FA setup

Get logged in user and create QR Code by generating a totp token. Encrypt the token with user email as encryption secret to bind the token with email. Send the encrypted token and QR to response.

```csharp
[HttpGet("GenerateAuthenticatorQrCode")]
public async Task<IActionResult> GenerateAuthenticatorQRCode()
{
    var currentUser = await GetAuthenticatedUser();

    if (currentUser == null || string.IsNullOrEmpty(currentUser.Email))
    {
        return BadRequest("User not found or email is missing.");
    }

    var secretKey = Base32Encoding.ToString(KeyGeneration.GenerateRandomKey(20));
    string appIssuer = Uri.EscapeDataString(ConfigurationManager.AppSettings["AppIssuer"]);
    string userEmail = Uri.EscapeDataString(currentUser.Email);
    string otpAuthUri = $"otpauth://totp/{appIssuer}:{userEmail}?secret={secretKey}&issuer={appIssuer}&algorithm=SHA1&digits=6&period=30";

    using var qrCodeGenerator = new QRCodeGenerator();
    using var qrData = qrCodeGenerator.CreateQrCode(otpAuthUri, QRCodeGenerator.ECCLevel.Q);
    using var qrCode = new PngByteQRCode(qrData);

    string qrBase64String = Convert.ToBase64String(qrCode.GetGraphic(20));
    string qrImage = $"data:image/png;base64,{qrBase64String}";

    var encryptedKey = EncryptionHelper.Encrypt(secretKey, currentUser.Email);

    return Ok(new AuthenticatorQRCodeResponse
    {
        Base64Image = qrImage,
        SecretKey = encryptedKey,
    });
}
```

### 2. Update User Authenticator

Get encrypted token and OTP from user via DTO of request. Get logged in user and decrypt the token by user email. Validate if provided OTP is valid. If valid, set the encrypted token to user data for future verifications.

```csharp
[HttpPut("UpdateAuthenticatorForUser")]
public async Task<IActionResult> UpdateAuthenticatorForUser(AuthenticatorUpdateRequest request)
{
    var currentUser = await GetAuthenticatedUser();

    if (currentUser == null || string.IsNullOrEmpty(currentUser.Email))
    {
        return BadRequest("User not found or email is missing.");
    }

    var decryptedKey = EncryptionHelper.Decrypt(request.SecretKey, currentUser.Email);

    if (!IsOtpValid(decryptedKey, request.OTP))
    {
        return BadRequest("Invalid OTP.");
    }

    currentUser.AuthenticatorKey = decryptedKey;
    _unitOfWork.Users.Update(currentUser);
    await _unitOfWork.SaveChangesAsync();

    return Ok(true);
}
```

### 3. Login User API

Check if user credentials are ok and create a temporary token with user id and send as response for next OTP verification step.

```csharp
[AllowAnonymous]
[HttpPost("LoginUser")]
public async Task<IActionResult> LoginUser(UserLoginRequest request)
{
    #region User Authentication Logic
    // User authentication logic (e.g., check credentials) goes here
    #endregion

    var authenticatedUser = await GetUserByCredentials(request);

    if (authenticatedUser == null)
    {
        return Unauthorized("Invalid credentials.");
    }

    var encryptedUserSecret = EncryptionHelper.Encrypt(authenticatedUser.Id.ToString(), ConfigurationManager.AppSettings["UserSigningKey"]);

    return Ok(new AuthenticationResponse { EncryptedOtpSecret = encryptedUserSecret });
}
```

### 4. User Login With OTP

Check if user credentials are ok and create a temporary token with user id and send as response for next OTP verification step.

```csharp
[AllowAnonymous]
[HttpPost("LoginWithOtp")]
public async Task<IActionResult> LoginWithOtp(LoginWithOtpRequest request)
{
    #region Validate DTO
    // Validate incoming request DTO (e.g., check required fields)
    #endregion

    var decryptedUserId = EncryptionHelper.Decrypt(request.EncryptedOtpSecret, ConfigurationManager.AppSettings["UserSigningKey"]);
    if (!long.TryParse(decryptedUserId, out long userId))
        return BadRequest("Invalid user identifier.");

    var user = await _unitOfWork.Users.GetById(userId);
    if (user == null)
        return Unauthorized("User not found.");

    var isValidOtp = IsOtpValid(user.AuthenticatorKey, request.OTP);
    if (!isValidOtp)
        return BadRequest("Invalid OTP.");

    var (jwtToken, jwtExpiry) = GenerateJwtToken(user);
    var (refreshToken, refreshExpiry) = GenerateRefreshToken();

    var tokenResponse = new UserTokenResponse
    {
        JwtToken = jwtToken,
        JwtExpiresAt = jwtExpiry,
        RefreshToken = refreshToken,
        RefreshExpiresAt = refreshExpiry
    };

    await _unitOfWork.Tokens.Save(tokenResponse);
    return Ok(tokenResponse);
}
```

### Is Authenticator OTP Valid Helper Methods

Checks the provided OTP against a secret token and returns boolean response. Allowed two future windows to mitigate issues with server and client system clock slight mismatches.

```csharp
private static bool IsOtpValid(string secretKey, string otp)
{
    if (string.IsNullOrEmpty(secretKey) || string.IsNullOrEmpty(otp))
        return false;

    var totp = new Totp(Base32Encoding.ToBytes(secretKey));
    return totp.VerifyTotp(otp, out _, new VerificationWindow(previous: 0, future: 2));
}
```


## Conclusion

Integrating TOTP-based 2FA with QR codes in a .NET application improves security. By leveraging Otp.NET for generating TOTP codes and QRCoder for creating QR codes, users can securely authenticate using Google Authenticator, Microsoft Authenticator, or similar apps. Implement this setup to enhance your application's security and safeguard user accounts effectively.
