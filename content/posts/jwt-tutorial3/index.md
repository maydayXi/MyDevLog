---
date: "2025-04-04T15:32:58+08:00"
lastmod: "2025-04-06 22:14:08"
draft: false
title: "ASP.NET Core JWT 驗證進階"
description: "錯誤處理與 Unit Of Work，提升系統穩定性及資料一致性"
slug: asp-net-core-jwt-error-handling-uow
categories:
  - Web API
tags:
  - ASP.NET Core
  - C#
  - .NET 8
  - JWT
  - JSON Web Token
  - Refresh Token
  - Authentication
  - Unit Of Work
  - Exception Handle
image: "exception-featured.svg"
links:
  - title: JWT Authentication with .NET 9
    description: Youtube 教學影片
    website: https://www.youtube.com/watch?v=6EEltKS8AwA
    image: https://cdn.jsdelivr.net/gh/maydayXi/MyDevLog@main/content/posts/jwt-tutorial/yt-icon.png
  - title: JWT-Authentication-API
    description: 本專案原始碼
    website: https://github.com/maydayXi/JWT-Authentication-API
    image: https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png
---

# 序

原本預計 ASP.NET Core JWT 分成兩篇寫完，不過因為自己也是第一次實作，再加上不想完全照著教學的影片，過於簡單，與實務上還有一段差別，所以我加入了很多我自己在工作上的經驗跟作法，沒想到導致篇幅內容增加，只好分成第三篇，將最後需要加強與改進的部分寫在這一篇中！

# 本篇重點

1. **Validation：針對進入資料庫的資料進行驗證**
2. **Unit Of Work：確保資料的一致性**
3. **Exception Handle：統一的錯誤處理機制**，方便讓前端在呼叫 API 的時候可以知道系統或相關錯誤。

# 資料驗證

目前程式架構及資料輸入流如下

- 使用者從瀏覽器發送請求後，到 Controller 接收會包裝成 Dto 的物件型別，這時候需要做**第一層的驗證，驗證使用者的請求，如果有問題，回傳 Http 400**
- 較嚴格的系統可能會在 Controller 傳送 Dto 到 Service 的時候作**第二層的驗證，如果這一層有問題統一丟出 `Exception`，屬於伺服器的錯誤回傳 Http 500**

```
 ┌──────┐          ┌──────────┐            ┌───────┐       ┌──────┐
 │Client│          │Controller│            │Service│       │Entity│
 └──┬───┘          └────┬─────┘            └───┬───┘       └──┬───┘
    │                   │                      │              │
    │Request from client│                      │              │
    │──────────────────>│                      │              │
    │                   │                      │              │
    │                   │[Data Transfer Object]│              │
    │                   │─────────────────────>│              │
    │                   │                      │              │
    │                   │                      │[Entity class]│
    │                   │                      │─────────────>│
 ┌──┴───┐          ┌────┴─────┐            ┌───┴───┐       ┌──┴───┐
 │Client│          │Controller│            │Service│       │Entity│
 └──────┘          └──────────┘            └───────┘       └──────┘
```

整個應用程式共有四個 API，分析如下

1. **註冊**：需要**驗證註冊資料**
2. **登入**：需要**驗證登入資料**，登入時需要產生 RefreshToken 給員工，所以**還要再驗證 RefreshToken 及所屬員工資料**。
3. **登出**：會從 Client 收到 **AccessToken 及 RefreshToken，前者從 Header 讀取，後者從 Body 讀取**，因為要將 AccessToken 及 RefreshToken 寫入黑名單，兩者都是 token，所以**只要驗證 token**
4. **更新**：會從 **Body 收到 RefreshToken，檢查有沒有過期**，確認沒有過期後由程式產生一組新的 AccessToken 及 RefreshToken，而**員工可以從 RefreshToken 的關聯參考找到這個 RefreshToken 屬於哪一個員工**，進而取得員工資料，**所以一樣只需要驗證 Token**

## 註冊、登入

其中註冊、登入都會驗證使用者的帳號跟密碼，所以將帳號密碼的驗證寫在一起，可以簡化程式，不用在註冊時寫一次，在登入時在寫一次同樣的驗證邏輯，而且將驗證邏輯抽離，讓 Controller 責任更專一

由於 `RegisterDto` 及 `LoginDto` 兩個類別的欄位是一樣的，我為可程式的可讀性才分成兩個，所以本質上要驗證的欄位都一樣，我要新增一個驗證員工用的 interface `IEmployeeCredentialDto.cs` 標示要驗證的欄位有哪些，如下

之所以加一個驗證用的介面是因為驗證的方法，參數我希望**可以使用 Generic 的型別**，也就是**可以傳入 `RegisterDto` 或 `LoginDto` 在同一個方法**，不然就要分成兩個方法，一個驗證 `RegisterDto`，另一個驗證 `LoginDto`，而且兩個方法的驗證邏輯是一樣的，重寫兩次的話會失去抽離的意義

```csharp
namespace JWT_Authentication_API.Interfaces;

/// <summary>
/// 員工驗證欄位介面
/// </summary>
public interface IEmployeeCredentialDto
{
    /// <summary>
    /// 員工帳號
    /// </summary>
    public string Email { get; }
    /// <summary>
    /// 員工密碼
    /// </summary>
    public string Password { get; }
}
```

接著讓 `RegisterDto`, `LoginDto` 實作

```csharp
using JWT_Authentication_API.Interfaces;

namespace JWT_Authentication_API.Models;

/// <summary>
/// 使用者登入的資料模型
/// </summary>
public class LoginDto : IEmployeeCredentialDto
{
    /// <summary>
    /// 使用者帳號
    /// </summary>
    public string Email { get; set; } = string.Empty;
    /// <summary>
    /// 使用者密碼（未加密）
    /// </summary>
    public string Password { get; set; } = string.Empty;
}

using JWT_Authentication_API.Interfaces;

namespace JWT_Authentication_API.Models;

/// <summary>
/// 註冊使用者的資料模型
/// </summary>
public class RegisterDto : IEmployeeCredentialDto
{
    /// <summary>
    /// 使用者帳號
    /// </summary>
    public string Email { get; set; } = string.Empty;
    /// <summary>
    /// 使用者密碼（未加密）
    /// </summary>
    public string Password { get; set; } = string.Empty;
}
```

### CredentialHelper

接下來新增驗證用的 Helper，一樣是用 `static` 的形式，因為只負責登入及註冊的身份驗證，不需要跟資料庫互動如下，新增 `CredentialHelper.cs` 並實作驗證的邏輯，新增 `IsValidEmployeeCredential` 方法，**因為教學的原故，我只驗證有沒有值而已，實際上還會驗證 Email 的格式，密碼的長度、複雜度**…等，相關的驗證邏輯都可以新增在這裡

```csharp
using JWT_Authentication_API.Interfaces;

namespace JWT_Authentication_API.Helper;

/// <summary>
/// 身分驗證的輔助類別
/// </summary>
public static class CredentialHelper
{
    /// <summary>
    /// 驗證員工的帳號及密碼
    /// </summary>
    /// <param name="dto"> 要驗證的員工資料 </param>
    /// <typeparam name="T"> 註冊及登入的介面 </typeparam>
    /// <returns> 有沒有通過驗證 </returns>
    public static bool IsValidEmployeeCredential<T>(T dto)
        where T : IEmployeeCredentialDto
    {
        return !string.IsNullOrEmpty(dto.Email) && !string.IsNullOrEmpty(dto.Password);
    }
}
```

最後回到 `AuthController` 將有驗證的地方修改如下

```csharp
#region 註冊
/// <summary>
/// 註冊 API
/// </summary>
/// <param name="registerDto"> 使用者傳送的員工註冊資料 </param>
/// <returns> 註冊結果 </returns>
[AllowAnonymous]
[HttpPost("register")]
public async Task<ActionResult> RegisterAsync(RegisterDto registerDto)
{
    // 驗證註冊資料
    if(!CredentialHelper.IsValidEmployeeCredential(registerDto))
        return BadRequest("Please provide 'Email' and 'Password'");

    // 先查詢有沒有重複的員工資料
    var employee = await _employeeService.GetEmployeeByEmailAsync(registerDto.Email);

    // 如果沒有就註冊新員工
    var registerResult =
        employee == null &&
        await _employeeService.AddEmployeeAsync(registerDto);

    // 回傳註冊結果
    ......
}
#endregion

#region 登入
/// <summary>
/// 登入 API
/// </summary>
/// <param name="loginDto"> 使用者的輸入資料 </param>
/// <returns> 登入結果 </returns>
[AllowAnonymous]
[HttpPost("login")]
public async Task<ActionResult> LoginAsync(LoginDto loginDto)
{
    // 登入資料驗證
    if (!CredentialHelper.IsValidEmployeeCredential(loginDto))
        return BadRequest("Please provide 'Email' and 'Password'");

    .....
}
#endregion
```

然後可以發現，在「**註冊**」方法中，有使用 2 個服務的方法，分別**查詢員工有沒有被註冊過，及新增員工資料**，一樣在這兩個方法加入驗證

### AddEmployeeAsync

開啟 `EmployeeService.cs`，修改 `AddEmployeeAsync` 方法，加入驗證的方法

```csharp
/// <summary>
/// 新增員工資料
/// </summary>
/// <param name="registerDto"> 員工註冊資料 </param>
/// <returns> 註冊結果 </returns>
public async Task<bool> AddEmployeeAsync(RegisterDto registerDto)
{
    if (!CredentialHelper.IsValidEmployeeCredential(registerDto))
        throw new ValidationException("Email or password is required");

    ......
}
```

### GetEmployeeByEmailAsync

由於這個方法只要驗證員工信箱，因此在 **_[CredentialHelper](#credentialhelper)_** 新增只驗證員工信箱的方法 **`IsValidEmail`** 如下，可以同步修改 `IsValidEmployeeCredential`

**這裡只驗證有沒有值，實務上企業內部會使用員工編號或英文姓名作為員工信箱，會有相關的驗證規則**

```csharp
/// <summary>
/// 驗證員工的帳號及密碼
/// </summary>
/// <param name="dto"> 要驗證的員工資料 </param>
/// <typeparam name="T"> 註冊及登入的介面 </typeparam>
/// <returns> 有沒有通過驗證 </returns>
public static bool IsValidEmployeeCredential<T>(T dto)
    where T : IEmployeeCredentialDto
{
    // 員工信箱及密碼的驗證規則，只檢查有沒有值
    return IsValidEmail(dto.Email) && !string.IsNullOrEmpty(dto.Password);
}

/// <summary>
/// 驗證員工信箱
/// </summary>
/// <param name="email"> 要驗證的員工信箱 </param>
/// <returns> 驗證結果 </returns>
public static bool IsValidEmail(string email)
{
    // Email 的驗證規則，這裡驗證是否有 Email 的值
    return !string.IsNullOrEmpty(email);
}
```

接著在 `GetEmployeeByEmailAsync` 方法加入 Email 驗證的方法如下

```csharp
/// <summary>
/// 依帳號取得員工資料
/// </summary>
/// <param name="email"> 登入信箱/註冊信箱 </param>
/// <returns> 員工資料 或 null </returns>
public async Task<EmployeeDto?> GetEmployeeByEmailAsync(string email)
{
    if(!CredentialHelper.IsValidEmail(email))
        throw new ArgumentException("Email is required!");

    ......
}
```

### AddRefreshTokenAsync

另外在登入方法有使用 **新增 RefreshToken 的方法，一樣要加入驗證**

由最一開始的分析可以知道，**除了登入，另外登出及更新方法，都有驗證 RefreshToken 的相關邏輯** 這邊為了統一驗證物件及簡化程式，要將這這方法重構，**將原本方法的參數改為 `RefreshTokenDto`**，在 `Models` 新增 `RefreshTokenDto.cs` 如下

```csharp
namespace JWT_Authentication_API.Models;

/// <summary>
/// RefreshToken 資料物件
/// </summary>
public class RefreshTokenDto
{
    /// <summary>
    /// 員工識別
    /// </summary>
    public Guid EmployeeId { get; set; } = Guid.Empty;
    /// <summary>
    /// RefreshToken
    /// </summary>
    public string RefreshToken { get; set; } = string.Empty;
}
```

接著開啟 `ITokenService.cs` 及 `TokenService.cs`，**修改 `AddRefreshTokenAsync` 方法接收參數為 `RefreshTokenDto`** 如下

```csharp
using JWT_Authentication_API.Models;

namespace JWT_Authentication_API.Interfaces;

/// <summary>
/// Token 相關服務的介面
/// </summary>
public interface ITokenService
{
    ......

    /// <summary>
    /// 新增 RefreshToken
    /// </summary>
    /// <param name="refreshToken"> 要新增的 RefreshToken </param>
    /// <returns> 新增的結果 </returns>
    Task<bool> AddRefreshTokenAsync(RefreshTokenDto refreshToken);

    ......
}

using JWT_Authentication_API.Entities;
using JWT_Authentication_API.Helper;
using JWT_Authentication_API.Interfaces;
using JWT_Authentication_API.Models;
using Microsoft.EntityFrameworkCore;

namespace JWT_Authentication_API.Services;
/// <summary>
/// Token 資料存取服務
/// </summary>
/// <param name="context"> 資料庫物件 </param>
public class TokenService(AppDbContext context): ITokenService
{
    /// <summary>
    /// 資料庫物件
    /// </summary>
    private readonly AppDbContext _appDb = context;

    ......

    #region Add Refresh Token
    /// <summary>
    /// 新增 RefreshToken
    /// </summary>
    /// <param name="refreshTokenDto"> 要新增的 RefreshToken </param>
    /// <returns> 新增的結果 </returns>
    public async  Task<bool> AddRefreshTokenAsync(RefreshTokenDto refreshTokenDto)
    {
        ......
    }
    #endregion
}
```

再開啟 `TokenHelper.cs` **新增 `IsValidRefreshToken` 方法，用來驗證 `RefreshTokenDto` 物件**如下

```csharp
using System.Security.Cryptography;
using JWT_Authentication_API.Models;

namespace JWT_Authentication_API.Helper;

/// <summary>
/// Token 輔助類別，宣告為 static
/// 主要負責產生及驗證
/// 不需要 DI 可以直接使用
/// </summary>
public static class TokenHelper
{
    ......

    /// <summary>
    /// 驗證 RefreshTokenDto
    /// </summary>
    /// <param name="refreshTokenDto"> 要驗證的 RefreshTokenDto </param>
    /// <returns> 驗證結果 </returns>
    public static bool IsValidRefreshToken(RefreshTokenDto refreshTokenDto)
    {
      return !string.IsNullOrEmpty(refreshTokenDto.RefreshToken) &&
             !refreshTokenDto.EmployeeId.Equals(Guid.Empty);
    }

    ....
}
```

回到 `TokenService.cs` 在 **`AddRefreshTokenAsync` 加入上面的驗證方法並重構** 如下

```csharp
#region Add Refresh Token
/// <summary>
/// 新增 RefreshToken
/// </summary>
/// <param name="refreshTokenDto"> 要新增的 RefreshToken </param>
/// <returns> 新增的結果 </returns>
public async  Task<bool> AddRefreshTokenAsync(RefreshTokenDto refreshTokenDto)
{
    // 驗證要新增的 RefreshToken 物件
    if(!TokenHelper.IsValidRefreshToken(refreshTokenDto))
        throw new ArgumentException(
            $"'{nameof(refreshTokenDto.RefreshToken)}' and '{nameof(refreshTokenDto.EmployeeId)}' is required!");
    // 新增 RefreshToken
    await _appDb.RefreshTokens.AddAsync(new RefreshToken
    {
        Token = refreshTokenDto.RefreshToken,
        // 配合週休二日，設定 5 天後到期
        ExpiresAt = DateTimeOffset.Now.AddDays(5),
        EmployeeId = refreshTokenDto.EmployeeId
    });
    // 新增結果
    return await _appDb.SaveChangesAsync() > 0;
}
```

最後回到 `AuthController` 登入方法，修改 `AddRefreshTokenAsync` 傳入的參數

```csharp
#region 登入
/// <summary>
/// 登入 API
/// </summary>
/// <param name="loginDto"> 使用者的輸入資料 </param>
/// <returns> 登入結果 </returns>
[AllowAnonymous]
[HttpPost("login")]
public async Task<ActionResult> LoginAsync(LoginDto loginDto)
{
    // 登入資料驗證
    if (!CredentialHelper.IsValidEmployeeCredential(loginDto))
        return BadRequest("Please provide 'Email' and 'Password'");

    ......

    var result = await _tokenService.AddRefreshTokenAsync(new RefreshTokenDto
    {
        EmployeeId = employee.Id,
        RefreshToken = refreshToken,
    });
    if(!result) return StatusCode(StatusCodes.Status500InternalServerError);

    ......
}
#endregion
```

## 登出

由一開始的分析可以知道登出要驗證 **Token（AccessToken 加入黑名單並刪除 RefreshToken）**，可以利用先前就建立好的 `TokenHelper`，**新增一個 `HasToken` 方法，接收一個 `token` 做為要驗證的 token 參數**

```csharp
#region Validation
/// <summary>
/// 驗證 JWT
/// </summary>
/// <param name="token"> JWT </param>
/// <returns> 驗證結果 </returns>
public static bool HasToken(string token)
{
    return !string.IsNullOrEmpty(token);
}
#endregion
```

回到 `AuthController` 對登出方法做重構，將參數改成跟登入一樣 `RefreshTokenDto`，然後加入上面的驗證方法如下

```csharp
#region 登出
/// <summary>
/// 登出 API
/// </summary>
/// <param name="refreshTokenDto"> 要刪除的 RefreshToken </param>
/// <returns> 登出結果 </returns>
[Authorize]
[HttpPost("logout")]
public async Task<IActionResult> LogoutAsync(RefreshTokenDto refreshTokenDto)
{
    // 從 header 讀取 JWT(AccessToken) 並進行驗證
    var token = $"{HttpContext.Request.Headers.Authorization}"
        .Replace("Bearer ", string.Empty, StringComparison.OrdinalIgnoreCase);

    if (!TokenHelper.HasToken(token))
        return BadRequest("'AccessToken' is required!");
    ......
}
#endregion
```

**但這樣的作法其實是比較嚴格的，因為如果沒有提供 Bearer Token 在 [Authorize] 的時候就會被擋下來了**

### AddTokenToTokenBlackListAsync

登出的時候會將 token 加入黑名單，這裡也加入驗證

```csharp
#region Add into BlackList
/// <summary>
/// 新增 Token 到黑名單
/// </summary>
/// <param name="token"> 要新增的 Token </param>
/// <returns> 新增結果 </returns>
/// <exception cref="ArgumentException"> token 是空值的時候 </exception>
public async Task<bool> AddTokenToTokenBlackListAsync(string token)
{
    // 檢查有沒有提供 token
    if(!TokenHelper.HasToken(token))
        throw new ArgumentException("'Access token' is required!");
    // 新增到黑名單
    await _appDb.TokenBlackLists.AddAsync(new TokenBlackList { Token = token });
    // 新增結果
    return await _appDb.SaveChangesAsync() > 0;
}
#endregion
```

### RemoveRefreshTokenAsync

最後登出會刪除 RefreshToken，同步進行重構，開啟 `ITokenService.cs` 及 `TokenService.cs`，修改 `RemoveRefreshTokenAsync` 方法，**將傳入的參數改成 `RefreshTokenDto`，並加入 Token 的驗證及重構** 如下

```csharp
using JWT_Authentication_API.Models;

namespace JWT_Authentication_API.Interfaces;

/// <summary>
/// Token 相關服務的介面
/// </summary>
public interface ITokenService
{
    ......

    /// <summary>
    /// 刪除 RefreshToken
    /// </summary>
    /// <param name="refreshTokenDto"> 要刪除的 RefreshToken </param>
    /// <returns> 刪除結果 </returns>
    Task<bool> RemoveRefreshTokenAsync(RefreshTokenDto refreshTokenDto);
}

using JWT_Authentication_API.Entities;
using JWT_Authentication_API.Helper;
using JWT_Authentication_API.Interfaces;
using JWT_Authentication_API.Models;
using Microsoft.EntityFrameworkCore;

namespace JWT_Authentication_API.Services;
/// <summary>
/// Token 資料存取服務
/// </summary>
/// <param name="context"> 資料庫物件 </param>
public class TokenService(AppDbContext context): ITokenService
{
    ......

    #region Remove Refresh Token
    /// <summary>
    /// 刪除 RefreshToken
    /// </summary>
    /// <param name="refreshTokenDto"> 要刪除的 RefreshToken </param>
    /// <returns> 刪除結果 </returns>
    /// <exception cref="KeyNotFoundException"> 找不到要刪除的 RefreshToken </exception>
    /// <exception cref="InvalidOperationException"> 舊的 RefreshToken 新增黑名單失敗 </exception>
    public async Task<bool> RemoveRefreshTokenAsync(RefreshTokenDto refreshTokenDto)
    {
        if (!TokenHelper.HasToken(refreshTokenDto.RefreshToken))
            throw new ArgumentException($"'{nameof(refreshTokenDto.RefreshToken)}' is required!");

        // 先找出要刪除的 refreshToken
        var refreshToken = await _appDb.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == refreshTokenDto.RefreshToken);

        // 找不到要刪除的 RefreshToken
        if (refreshToken is null)
            throw new KeyNotFoundException($"{nameof(refreshTokenDto.RefreshToken)} not found!");

        // 將要刪除的 RefreshToken 新增到黑名單
        var addResult = await AddTokenToTokenBlackListAsync(refreshToken.Token);
        if(!addResult) throw new InvalidOperationException
            ("Failed to add refresh token into black list!");

        // 再將舊的 RefreshToken 刪除
        _appDb.RefreshTokens.Remove(refreshToken);

        return await _appDb.SaveChangesAsync() > 0;
    }
    #endregion
}
```

### IsTokenRevokedAsync

最後要幫登出加上確認 Token 有沒有在黑名單，避免使用者傳送已經用過的 Token，開啟 `ITokenService.cs` 及 `TokenService.cs` 加入確認 Token 在黑名單的方法 `IsTokenRevokedAsync`

```csharp
using JWT_Authentication_API.Models;

namespace JWT_Authentication_API.Interfaces;

/// <summary>
/// Token 相關服務的介面
/// </summary>
public interface ITokenService
{
    ......

    /// <summary>
    /// 檢查 Token 有沒有被使用過
    /// </summary>
    /// <param name="token"> 要檢查的 Token </param>
    /// <returns> 檢查結果 </returns>
    Task<bool> IsTokenRevokedAsync(string token);

    ......
}

using JWT_Authentication_API.Entities;
using JWT_Authentication_API.Helper;
using JWT_Authentication_API.Interfaces;
using JWT_Authentication_API.Models;
using Microsoft.EntityFrameworkCore;

namespace JWT_Authentication_API.Services;
/// <summary>
/// Token 資料存取服務
/// </summary>
/// <param name="context"> 資料庫物件 </param>
public class TokenService(AppDbContext context): ITokenService
{
    ......

    #region Check Black list
    /// <summary>
    /// Token 是否被使用過
    /// </summary>
    /// <param name="token"> 要檢查的 Token </param>
    /// <returns> 結果 </returns>
    public async Task<bool> IsTokenRevokedAsync(string token)
    {
        return await _appDb.TokenBlackLists.AsNoTracking()
            .AnyAsync(l => l.Token == token);
    }
    #endregion

    ......
}
```

最後在 `LogoutAsync` 方法加入驗證邏輯如下

```csharp
#region 登出
/// <summary>
/// 登出 API
/// </summary>
/// <param name="refreshTokenDto"> 要刪除的 RefreshToken </param>
/// <returns> 登出結果 </returns>
[Authorize]
[HttpPost("logout")]
public async Task<IActionResult> LogoutAsync(RefreshTokenDto refreshTokenDto)
{
    // 從 header 讀取 JWT(AccessToken) 並進行驗證
    var token = $"{HttpContext.Request.Headers.Authorization}"
        .Replace("Bearer ", string.Empty, StringComparison.OrdinalIgnoreCase);

    ......

    // 檢查 AccessToken 有沒有被使用過
    if(await _tokenService.IsTokenRevokedAsync(token))
        return Unauthorized($"'{token}' is revoked!");

    ......
}
#endregion
```

## 更新

最後一個是更新 RefreshToken 的方法，從分析得知一樣**只要驗證 Token**，所以要將這個方法進行重構，**將這個方法接收的參數改成 `RefreshTokenDto` 並加入驗證方法 `TokenHelper.HasToken`** 如下

```csharp
#region Refresh
/// <summary>
/// 驗證、更新 RefreshToken
/// </summary>
/// <param name="refreshTokenDto"> 要驗證的 RefreshToken </param>
/// <returns> 新的 RefreshToken 或 AccessToken </returns>
[HttpPost("refresh-token")]
public async Task<ActionResult> RefreshTokenAsync([FromBody] RefreshTokenDto refreshTokenDto)
{
    // 驗證 RefreshToken
    if(!TokenHelper.HasToken(refreshTokenDto.RefreshToken))
        return BadRequest($"'{nameof(refreshTokenDto.RefreshToken)}' is required!");
    ......
}
```

接著需要將原本接收參數的方法都作修改

### IsRefreshTokenExpiredAsync

首先是檢查 RefreshToken 有沒過期的方法，開啟 `ITokenService.cs` 及 `TokenService.cs`，將**接收參數改成 `RefreshTokenDto`**，再將 **`IsRefreshTokenExpiredAsync` 方法加入驗證重構**如下

```csharp
/// <summary>
/// Token 相關服務的介面
/// </summary>
public interface ITokenService
{
    ......

    /// <summary>
    /// 檢查 RefreshToken 是否過期
    /// </summary>
    /// <param name="refreshToken"> 要檢查的 RefreshToken 物件 </param>
    /// <returns> 檢查結果 </returns>
    Task<bool> IsRefreshTokenExpiredAsync(RefreshTokenDto refreshToken);

    ......
}

using JWT_Authentication_API.Entities;
using JWT_Authentication_API.Helper;
using JWT_Authentication_API.Interfaces;
using JWT_Authentication_API.Models;
using Microsoft.EntityFrameworkCore;

namespace JWT_Authentication_API.Services;
/// <summary>
/// Token 資料存取服務
/// </summary>
/// <param name="context"> 資料庫物件 </param>
public class TokenService(AppDbContext context): ITokenService
{
    ......

    #region Validate RefreshToken
    /// <summary>
    /// 檢查 RefreshToken 是否過期
    /// </summary>
    /// <param name="refreshTokenDto"> 要檢查的 RefreshToken </param>
    /// <returns> 檢查結果 </returns>
    /// <exception cref="KeyNotFoundException"> 找不到要檢查的 RefreshToken </exception>
    public async Task<bool> IsRefreshTokenExpiredAsync(RefreshTokenDto refreshTokenDto)
    {
        // 驗證 RefreshToken
        if(!TokenHelper.HasToken(refreshTokenDto.RefreshToken))
            throw new ArgumentException($"'{nameof(refreshTokenDto.RefreshToken)}' is required!");

        // 取得要檢查的 RefreshToken
        var refreshToken = await _appDb.RefreshTokens.AsNoTracking()
            .FirstOrDefaultAsync(rt => rt.Token == refreshTokenDto.RefreshToken);

        // 找不到要檢查的 RefreshToken
        if (refreshToken is null)
            throw new KeyNotFoundException($"'{refreshTokenDto.RefreshToken}' not found!");

        // 回傳是否過期
        return DateTimeOffset.UtcNow > refreshToken.ExpiresAt;
    }
    #endregion

    ......
}
```

回到 `AuthController` 的更新方法 `RefreshTokenAsync` 中，將檢查過期的程式修改如下

```csharp
// 驗認 RefreshToken 是否過期了
if(await _tokenService.IsRefreshTokenExpiredAsync(refreshTokenDto))
    return BadRequest($"'{nameof(refreshTokenDto.RefreshToken)}' has expired!");
```

### UpdateRefreshTokenAsync

修改更新 RefreshToken 的方法，開啟 `ITokenService.cs` 及 `TokenService.cs`，**將 `UpdateRefreshTokenAsync` 方法接收參數改成 `RefreshTokenDto`，回傳值改為 `RefreshTokenDto?`**

接著**修改 `UpdateRefreshTokenAsync` 的實作並加入驗證方法**如下

```csharp

/// <summary>
/// Token 相關服務的介面
/// </summary>
public interface ITokenService
{
    ......

    /// <summary>
    /// 更新 RefreshToken
    /// </summary>
    /// <param name="refreshTokenDto"> 要更新的 RefreshToken </param>
    /// <returns> 更新後的 RefreshToken </returns>
    Task<RefreshTokenDto?> UpdateRefreshTokenAsync(RefreshTokenDto refreshTokenDto);

    ......
}

using JWT_Authentication_API.Entities;
using JWT_Authentication_API.Helper;
using JWT_Authentication_API.Interfaces;
using JWT_Authentication_API.Models;
using Microsoft.EntityFrameworkCore;

namespace JWT_Authentication_API.Services;
/// <summary>
/// Token 資料存取服務
/// </summary>
/// <param name="context"> 資料庫物件 </param>
public class TokenService(AppDbContext context): ITokenService
{

    ......

    #region Update Refresh Token
    /// <summary>
    /// 更新 RefreshToken, 將舊的 Token 寫入黑名單
    /// </summary>
    /// <param name="refreshTokenDto"> 要更新的 RefreshToken </param>
    /// <returns> 更新後的 RefreshToken </returns>
    /// <exception cref="KeyNotFoundException"> 找不到要更新的 RefreshToken </exception>
    /// <exception cref="InvalidOperationException"> 新增舊的 RefreshToken 黑名單失敗 </exception>
    public async Task<RefreshTokenDto?> UpdateRefreshTokenAsync(RefreshTokenDto refreshTokenDto)
    {
        // 驗證更新前的 RefreshToken
        if(!TokenHelper.HasToken(refreshTokenDto.RefreshToken))
            throw new ArgumentException($"'{nameof(refreshTokenDto.RefreshToken)}' is required!");

        // 先找出要更新的 refreshToken
        var refreshToken = await _appDb.RefreshTokens.FirstOrDefaultAsync(rt
            => rt.Token == refreshTokenDto.RefreshToken);

        // 如果找不到要刪除的 RefreshToken
        if (refreshToken is null) throw new KeyNotFoundException
            ($"'{nameof(refreshTokenDto.RefreshToken)}' not found!");

        // 將舊的 RefreshToken 新增到黑名單
        var addResult = await AddTokenToTokenBlackListAsync(refreshToken.Token);
        if(!addResult) throw new InvalidOperationException
            ($"Failed to add '{nameof(refreshTokenDto.RefreshToken)}' into black list!");

        // 更新 RefreshToken
        var newToken = TokenHelper.GenerateRefreshToken();
        refreshToken.Token = newToken;
        var result = await _appDb.SaveChangesAsync();

        // 更新失敗
        if (result != 1) throw new Exception("Refresh token update failed!");
        // 更新成功
        return await _appDb.RefreshTokens.AsNoTracking()
            .Select(rt => new RefreshTokenDto
            {
                EmployeeId = rt.EmployeeId,
                RefreshToken = rt.Token,
            })
            .FirstOrDefaultAsync(rt => rt.RefreshToken == newToken);
    }
    #endregion

    ......
}
```

回到 `AuthController` 的更新方法 `RefreshTokenAsync`，因為**將產生新的 RefreshToken 交給了 Service 去完成，所以將原本產生新的 RefreshToken 刪除並修改如下**

```csharp
// 更新 RefreshToken，並取得更新後的 RefreshToken
var result = await _tokenService.UpdateRefreshTokenAsync(refreshTokenDto);
// 更新成功但沒有回傳更新後的結果屬於伺服器錯誤
if (result is null) return StatusCode(StatusCodes.Status500InternalServerError);

// 找出目前使用的員工
var employee = await _employeeService.GetEmployeeByIdAsync(result.EmployeeId);
if (employee is null) return NotFound("Employee not found!");
```

更新跟登出一樣，需要再加上確認 Token 有沒有過期的驗證

```csharp
#region Refresh
/// <summary>
/// 驗證、更新 RefreshToken
/// </summary>
/// <param name="refreshTokenDto"> 要驗證的 RefreshToken </param>
/// <returns> 新的 RefreshToken 或 AccessToken </returns>
[HttpPost("refresh-token")]
public async Task<ActionResult> RefreshTokenAsync([FromBody] RefreshTokenDto refreshTokenDto)
{
    ......

    // 確認 Token 有沒有被使用過
    if(await _tokenService.IsTokenRevokedAsync(refreshTokenDto.RefreshToken))
        return Unauthorized($"'{refreshTokenDto.RefreshToken}' is revoked!");

    ......
}
```

基本上驗證的部分就重構完了，接下來可以使用 Postman 來測試，測試可以參考前一篇，在最下方的相關文章可以找到，這裡不再多描述了

## 資料驗證測試

特別提一下資料驗證的測試方法，因為**登出、更新**方法的接收參數變了，接收一個 `RefreshTokenDto` 物件，參考 **_[AddRefreshTokenAsync](#addrefreshtokenasync)_**，有一個 **EmployeeId, RefreshToken**，而 EmployeeId 提供預設值 **Guid.Empty**，所以在測試的時候可以只傳送 RefreshToken 就可以了

```json
{
  "refreshToken": "your-refresh-token"
}
```

# UnitOfWork

在測試中有一種情況，在登出的時候，用**已使用過的 RefreshToken 去登出的話，會出現「找不到 RefreshToken」的 Http 500**，結果看起來沒有錯，不過 **再次使用正確的 RefreshToken 登出的話，會出現「這個 AccessToken 出現在黑名單」** 中，這是因為在登出方法中，**先將 AccessToken 寫入黑名單中，再將 RefreshToken 寫入黑名單如下**，只要其中一個操作失敗了，兩個操作都要回到一開始的狀態。

**AccessToken 寫入黑名操作順利完成，不過 RefreshToken 寫入黑名單，因為找不到使用中的 RefreshToken 而失敗，理論上來說這一整個操作應該視為失敗**，要將原本寫入黑名單的 AccessToken 移除

```csharp
#region 登出
/// <summary>
/// 登出 API
/// </summary>
/// <param name="refreshTokenDto"> 要刪除的 RefreshToken </param>
/// <returns> 登出結果 </returns>
[Authorize]
[HttpPost("logout")]
public async Task<IActionResult> LogoutAsync(RefreshTokenDto refreshTokenDto)
{
    // 從 header 讀取 JWT(AccessToken) 並進行驗證
    var token = $"{HttpContext.Request.Headers.Authorization}"
        .Replace("Bearer ", string.Empty, StringComparison.OrdinalIgnoreCase);

    if (!TokenHelper.HasToken(token)) return BadRequest("'AccessToken' is required!");

    // 檢查 AccessToken 有沒有被使用過
    if(await _tokenService.IsTokenRevokedAsync(token)) return Unauthorized($"'{token}' is revoked!");

    // 將 JWT 加入黑名單
    var result = await _tokenService.AddTokenToTokenBlackListAsync(token);
    if (!result) return BadRequest("Logout failed!");

    // 將 refreshToken 刪除
    result = await _tokenService.RemoveRefreshTokenAsync(refreshTokenDto);
    if (!result) return BadRequest("Logout failed!");

    return Ok("Logout successfully!");
}
#endregion
```

而 UnitOfWork 就是為了解決這樣的問題而出現的，中文應該是「**工作單元**」，**每一個工作單元包含了所有完成這個方法所需要的資料操作，並且在所有操作完成後統一存儲**
