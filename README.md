# 我的世界插件 - 微软登录验证

这是一个基于Kotlin开发的我的世界插件项目，其主要目的是在玩家每次登录时，通过验证微软账户来确保只有经过授权的用户才能使用。

## 验证方式
### 验证机制
本插件采用严谨的验证流程，确保游戏账户的安全性。在玩家尝试登录我的世界服务器时，系统会引导玩家完成微软账户的登录流程。登录成功后，微软会返回一个唯一的访问令牌（Access Token），该令牌包含了用户的身份验证信息。

插件会将这个令牌与预先存储在数据库中的有效令牌进行比对。数据库使用 [数据库名称，可替换] 进行管理，采用 [加密方式，如 AES 加密] 对存储的令牌进行加密处理，防止数据泄露。只有当传入的令牌与数据库中的记录匹配，并且令牌处于有效期内时，玩家才能成功登录游戏。

### 技术实现

#### 示例代码
以下是一个简化的 Kotlin 代码示例，展示了如何集成微软认证和数据库验证：

```kotlin
import java.sql.DriverManager
import com.microsoft.aad.msal4j.ConfidentialClientApplication
import java.util.concurrent.CompletableFuture

object AuthService {
    private const val DB_URL = "jdbc:mysql://localhost:3306/minecraft_auth"
    private const val DB_USER = "root"
    private const val DB_PASSWORD = "password"

    private const val CLIENT_ID = "your-client-id"
    private const val CLIENT_SECRET = "your-client-secret"
    private const val AUTHORITY = "https://login.microsoftonline.com/your-tenant-id"

    suspend fun authenticate(token: String): Boolean {
        // 验证微软令牌
        val isValidMicrosoftToken = validateMicrosoftToken(token)
        if (!isValidMicrosoftToken) {
            return false
        }

        // 验证数据库
        return validateDatabaseToken(token)
    }

    private fun validateMicrosoftToken(token: String): Boolean {
        val app = ConfidentialClientApplication.builder(
            CLIENT_ID,
            com.microsoft.aad.msal4j.ClientCredentialFactory.createFromSecret(CLIENT_SECRET)
        ).authority(AUTHORITY).build()

        val future: CompletableFuture<com.microsoft.aad.msal4j.IAuthenticationResult> = app.acquireToken(
            com.microsoft.aad.msal4j.ClientCredentialParameters.builder(
                setOf("https://graph.microsoft.com/.default")
            ).build()
        )

        try {
            val result = future.get()
            return result.accessToken() == token
        } catch (e: Exception) {
            e.printStackTrace()
            return false
        }
    }

    private fun validateDatabaseToken(token: String): Boolean {
        DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD).use {
            val statement = it.prepareStatement("SELECT COUNT(*) FROM tokens WHERE token = ?")
            statement.setString(1, token)
            val resultSet = statement.executeQuery()
            if (resultSet.next()) {
                return resultSet.getInt(1) > 0
            }
        }
        return false
    }
}
```

#### 技术实现
- **微软认证集成**：使用微软官方提供的 [API 名称，如 Microsoft Identity Platform] 进行用户认证，确保符合微软的安全标准。
- **数据库交互**：通过 [数据库连接库名称，如 JDBC] 与数据库进行交互，执行令牌的查询和验证操作。
- **异常处理**：对认证过程中可能出现的异常情况，如网络错误、令牌过期、数据库连接失败等，进行了详细的错误处理和日志记录，方便管理员排查问题。