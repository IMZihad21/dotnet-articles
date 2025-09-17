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
[HttpGet("CreateProtectedQr")]
public async Task<IActionResult> CreateProtectedQr()
{
    // Retrieve the currently authenticated user
    var user = await GetCurrentUserAsync();

    if (user == null || string.IsNullOrWhiteSpace(user.UniqueId))
        return BadRequest("User not valid.");

    // Generate a random TOTP secret key
    var secretKey = Base32Encoding.ToString(KeyGeneration.GenerateRandomKey(20));

    // Prepare TOTP URI (used by authenticator apps)
    string serviceName = Uri.EscapeDataString(ConfigurationManager.AppSettings["ServiceIssuer"]);
    string totpUri = $"otpauth://totp/{serviceName}:{Uri.EscapeDataString(user.Email)}" +
                     $"?secret={secretKey}&issuer={serviceName}&algorithm=SHA1&digits=6&period=30";

    // Generate QR code as Base64 string
    string qrBase64 = GenerateQrCodeBase64(totpUri);

    // Encrypt the secret key for secure storage/transmission
    string encryptedKey = CryptoHelper.Encrypt(secretKey, user.UniqueId);

    // Return both QR code image and encrypted secret
    return Ok(new ProtectedQrResponse
    {
        QrCodeBase64 = qrBase64,
        EncryptedKey = encryptedKey
    });
}
```

### 2. Update User Authenticator

Get encrypted token and OTP from user via DTO of request. Get logged in user and decrypt the token by user email. Validate if provided OTP is valid. If valid, set the encrypted token to user data for future verifications.

```csharp
[HttpPut("UpdateUserAuthenticator")]
public async Task<IActionResult> UpdateUserAuthenticator(AuthenticatorUpdateRequest input)
{
    // Fetch the currently authenticated user
    var currentUser = await GetCurrentUserAsync();

    if (currentUser == null || string.IsNullOrWhiteSpace(currentUser.UniqueId))
        return BadRequest("User not valid.");

    // Decrypt the incoming secured key
    var decryptedKey = CryptoHelper.Decrypt(input.EncryptedKey, currentUser.UniqueId);

    // Validate the one-time password provided by the user
    if (!IsOtpValid(decryptedKey, input.OTP))
        return BadRequest("OTP verification failed.");

    // Update user's authenticator key and save changes
    currentUser.AuthKey = decryptedKey;
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
public async Task<IActionResult> LoginUser(UserLoginRequest input)
{
    #region User Verification
    // Validate user credentials against the database or identity store
    #endregion

    var user = await GetUserByCredentialsAsync(input);

    if (user == null)
        return Unauthorized("Credentials are invalid.");

    // Use a local variable for the signing key (source hidden)
    var secureSigningKey = "SomeSecureKeyValue"; // Hidden/internal key

    // Encrypt user identifier using the hidden key
    var encryptedIdentifier = CryptoHelper.Encrypt(user.Id.ToString(), secureSigningKey);

    return Ok(new UserLoginResponse
    {
        EncryptedOtpIdentifier = encryptedIdentifier
    });
}
```

### 4. User Login With OTP

Check if user credentials are ok and create a temporary token with user id and send as response for next OTP verification step.

```csharp
[AllowAnonymous]
[HttpPost("ValidateOtpLogin")]
public async Task<IActionResult> ValidateOtpLogin(OtpLoginInput input)
{
    #region Input Validation
    // Ensure all required fields are provided in the request
    #endregion

    // Hidden/internal signing key
    var secureSigningKey = "HiddenSecureKey";

    // Decrypt the encoded identifier from the request
    var decryptedId = CryptoHelper.Decrypt(input.EncryptedOtpIdentifier, secureSigningKey);
    if (!long.TryParse(decryptedId, out long userId))
        return BadRequest("Invalid user identifier.");

    // Retrieve user account
    var account = await _unitOfWork.Users.GetById(userId);
    if (account == null)
        return Unauthorized("User not found.");

    // Validate OTP
    if (!IsOtpValid(account.AuthKey, input.OTP))
        return BadRequest("Invalid OTP.");

    // Generate access token only
    var (accessToken, accessExpiry) = GenerateAccessToken(account);

    // Return access token in response (no token saving)
    return Ok(new UserSessionOutput
    {
        AccessToken = accessToken,
        AccessTokenExpiry = accessExpiry
    });
}
```

### Is Authenticator OTP Valid Helper Methods

Checks the provided OTP against a secret token and returns boolean response. Allowed two future windows to mitigate issues with server and client system clock slight mismatches.

```csharp
private static bool IsOtpValid(string authKey, string code)
{
    if (string.IsNullOrWhiteSpace(authKey) || string.IsNullOrWhiteSpace(code))
        return false;

    var totp = new Totp(Base32Encoding.ToBytes(authKey));

    // Return true if OTP is valid within the allowed window, otherwise false
    bool isValid = totp.VerifyTotp(code, out _, new VerificationWindow(previous: 2, future: 2));
    return isValid;
}
```


## Conclusion

Integrating TOTP-based 2FA with QR codes in a .NET application improves security. By leveraging Otp.NET for generating TOTP codes and QRCoder for creating QR codes, users can securely authenticate using Google Authenticator, Microsoft Authenticator, or similar apps. Implement this setup to enhance your application's security and safeguard user accounts effectively.
