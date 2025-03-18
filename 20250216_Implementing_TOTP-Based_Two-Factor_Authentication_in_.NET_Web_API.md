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
[HttpGet("GenerateSecureQrCode")]
public async Task<IActionResult> GenerateSecureQrCode()
{
    var activeUser = await RetrieveAuthenticatedUser();

    if (activeUser == null || string.IsNullOrEmpty(activeUser.Email))
    {
        return BadRequest("User not found or missing email.");
    }

    var secureKey = Base32Encoding.ToString(KeyGeneration.GenerateRandomKey(20));
    string issuer = Uri.EscapeDataString(ConfigurationManager.AppSettings["ServiceIssuer"]);
    string encodedEmail = Uri.EscapeDataString(activeUser.Email);
    string otpUri = $"otpauth://totp/{issuer}:{encodedEmail}?secret={secureKey}&issuer={issuer}&algorithm=SHA1&digits=6&period=30";

    using var qrGenerator = new QRCodeGenerator();
    using var qrData = qrGenerator.CreateQrCode(otpUri, QRCodeGenerator.ECCLevel.Q);
    using var qrCode = new PngByteQRCode(qrData);

    string base64QrCode = Convert.ToBase64String(qrCode.GetGraphic(20));
    string qrCodeImage = $"data:image/png;base64,{base64QrCode}";

    var securedKey = CryptoHelper.Encrypt(secureKey, activeUser.Email);

    return Ok(new SecureQrCodeResponse
    {
        QrImageBase64 = qrCodeImage,
        SecuredKey = securedKey,
    });
}
```

### 2. Update User Authenticator

Get encrypted token and OTP from user via DTO of request. Get logged in user and decrypt the token by user email. Validate if provided OTP is valid. If valid, set the encrypted token to user data for future verifications.

```csharp
[HttpPut("ModifyUserAuthenticator")]
public async Task<IActionResult> ModifyUserAuthenticator(AuthenticatorModifyRequest request)
{
    var activeUser = await RetrieveAuthenticatedUser();

    if (activeUser == null || string.IsNullOrEmpty(activeUser.Email))
    {
        return BadRequest("User not found or missing email.");
    }

    var unencryptedKey = CryptoHelper.Decrypt(request.SecuredKey, activeUser.Email);

    if (!ValidateOtp(unencryptedKey, request.OTP))
    {
        return BadRequest("Invalid OTP.");
    }

    activeUser.AuthKey = unencryptedKey;
    _unitOfWork.Users.Update(activeUser);
    await _unitOfWork.SaveChangesAsync();

    return Ok(true);
}
```

### 3. Login User API

Check if user credentials are ok and create a temporary token with user id and send as response for next OTP verification step.

```csharp
[AllowAnonymous]
[HttpPost("AuthenticateUser")]
public async Task<IActionResult> AuthenticateUser(UserAuthRequest request)
{
    #region Authentication Logic
    // Logic to verify user credentials
    #endregion

    var verifiedUser = await FetchUserByCredentials(request);

    if (verifiedUser == null)
    {
        return Unauthorized("Invalid credentials.");
    }

    var encodedUserSecret = CryptoHelper.Encrypt(verifiedUser.Id.ToString(), ConfigurationManager.AppSettings["SigningKey"]);

    return Ok(new AuthResponse { EncodedOtpKey = encodedUserSecret });
}
```

### 4. User Login With OTP

Check if user credentials are ok and create a temporary token with user id and send as response for next OTP verification step.

```csharp
[AllowAnonymous]
[HttpPost("VerifyOtpLogin")]
public async Task<IActionResult> VerifyOtpLogin(OtpLoginRequest request)
{
    #region Validate Input
    // Ensure all necessary request fields are present
    #endregion

    var decryptedUserId = CryptoHelper.Decrypt(request.EncodedOtpKey, ConfigurationManager.AppSettings["SigningKey"]);
    if (!long.TryParse(decryptedUserId, out long userId))
        return BadRequest("Invalid user identifier.");

    var userAccount = await _unitOfWork.Users.GetById(userId);
    if (userAccount == null)
        return Unauthorized("User not found.");

    var otpValidation = ValidateOtp(userAccount.AuthKey, request.OTP);
    if (!otpValidation)
        return BadRequest("Invalid OTP.");

    var (accessToken, tokenExpiry) = GenerateAccessToken(userAccount);
    var (refreshToken, refreshExpiry) = GenerateRefreshToken();

    var sessionResponse = new UserSessionResponse
    {
        AccessToken = accessToken,
        AccessTokenExpiry = tokenExpiry,
        RefreshToken = refreshToken,
        RefreshTokenExpiry = refreshExpiry
    };

    await _unitOfWork.Tokens.Save(sessionResponse);
    return Ok(sessionResponse);
}
```

### Is Authenticator OTP Valid Helper Methods

Checks the provided OTP against a secret token and returns boolean response. Allowed two future windows to mitigate issues with server and client system clock slight mismatches.

```csharp
private static bool ValidateOtp(string secretKey, string otpCode)
{
    if (string.IsNullOrEmpty(secretKey) || string.IsNullOrEmpty(otpCode))
        return false;

    var totpGenerator = new Totp(Base32Encoding.ToBytes(secretKey));
    return totpGenerator.VerifyTotp(otpCode, out _, new VerificationWindow(previous: 0, future: 2));
}
```


## Conclusion

Integrating TOTP-based 2FA with QR codes in a .NET application improves security. By leveraging Otp.NET for generating TOTP codes and QRCoder for creating QR codes, users can securely authenticate using Google Authenticator, Microsoft Authenticator, or similar apps. Implement this setup to enhance your application's security and safeguard user accounts effectively.
