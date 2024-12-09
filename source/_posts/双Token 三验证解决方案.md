---
title: 双Token解决方案
date: 2024-10-05 16:00:10
tags:
---

# 

以往的项目大部分解决方案为单 token：

+ 用户登录后，服务端颁发 jwt 令牌作为 token 返回
+ 每次请求，前端携带 token 访问，服务端解析 token 进行校验和鉴权

存在的问题：

+ 有效期设置问题：有效期设置需要对时间做平衡，不能太短也不能太长
+ 续期问题：一旦过期，用户必须重新登录，很难做无感刷新
+ 无状态问题：token 是无状态的，单 token 颁发后服务端无法主动使其失效

## 原理解析
---

这里引入双 token 机制：

+ accessToken：时间较短，一般为 1天
+ refreshToken：时间较长，一般为 3 天

登录过程：

+ 用户携带用户名和密码登录
+ 服务端为其颁发 accessToken 和 refreshToken

三验证环节：

+ 一验证：前端请求携带 accessToken，验证是否过期，不过期放行，过期则进入第二个验证环节
+ 二验证：前端请求携带 refreshToken，验证是否过期，不过期进入第三个验证环节，过期则要求用户重新登录
+ 三验证：对 ip 地址进行限流，对 refreshToken 解析 判断是否存在，存在则颁发新的 accessToken 和 refreshToken 返回前端更新，前端没接收成功或者失败则删除所有token，

##   
 生成 Token
```java
/**
     * 根据UserDetails生成对应的Jwt令牌
     * @param user 用户信息
     * @return 令牌
     */
    public String createJwt(UserDetails user, String username, int userId) {
        if(this.frequencyCheck(userId)) {
            Algorithm algorithm = Algorithm.HMAC256(key);
            Date expire = this.expireTime();
            return JWT.create()
                .withJWTId(UUID.randomUUID().toString())
                .withClaim("id", userId)
                .withClaim("name", username)
                .withClaim("authorities", user.getAuthorities()
                    .stream()
                    .map(GrantedAuthority::getAuthority).toList())
                .withExpiresAt(expire)
                .withIssuedAt(new Date())
                .sign(algorithm);
        } else {
            return null;
        }
    }
    // 生成refreshToken
    public String createRefreshJwt(UserDetails user, String username, int userId) {
        Algorithm algorithm = Algorithm.HMAC256(key);
        Date expire = this.reExpireTime();
        return JWT.create()
            .withJWTId(UUID.randomUUID().toString())
            .withClaim("id", userId)
            .withClaim("name", username)
            .withExpiresAt(expire)
            .withIssuedAt(new Date())
            .sign(algorithm);
    }
```

## 校验 Token
```java
  @Resource
    JwtUtils utils;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain) throws ServletException, IOException {
        // 获取请求头
        String authorization = request.getHeader("Authorization");
        // 从请求头解析jwt
        DecodedJWT jwt = utils.resolveJwt(authorization);
        if(jwt != null) {
            //开始解析成UserDetails对象，如果得到的是null说明解析失败，JWT有问题
            UserDetails user = utils.toUser(jwt);
            //验证没有问题，那么就可以开始创建Authentication了，这里我们跟默认情况保持一致
            //使用UsernamePasswordAuthenticationToken作为实体，填写相关用户信息进去
            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            //然后直接把配置好的Authentication塞给SecurityContext表示已经完成验证
            SecurityContextHolder.getContext().setAuthentication(authentication);
            // 将该用户jwt添加到请求头
            request.setAttribute(Const.ATTR_USER_ID, utils.toId(jwt));
        }
        // 触发http请求
        filterChain.doFilter(request, response);
    }
```

## 刷新 Token
```java
@Data
public class AuthorizeRespDTO {
    String username;
    String role;
    String access_token;
    String refresh_token;
    Date access_expire;
    Date refresh_expire;
}

```

```java
@Override
    public AuthorizeRespDTO refreshToken(String token, HttpServletRequest request) {
        String ip = request.getRemoteAddr();
        AuthorizeRespDTO dto = null;
        // 进行限流处理
        if (verifyLimit(ip)){
            String refreshToken = "Bearer " + token;
            // 如果 Access Token 无效，但存在 Refresh Token，尝试解析 Refresh Token
            DecodedJWT refreshJwt = utils.resolveJwt(refreshToken);
            // 如果没有过期进行下一步操作
            if (refreshJwt != null) {
                UserDetails user = utils.toRefreshUser(refreshJwt);
                if (user != null) {
                    // 将新的 Access Token 加到响应头中
                    AccountRespDTO account = findAccountByNameOrEmail(user.getUsername());
                    // 如果 Refresh Token 有效，生成新的 Access Token
                    String newAccessToken = utils.createJwt(user, account.getUsername(), account.getId()); // 你需要实现这个方法
                    String newRefreshJwt = utils.createRefreshJwt(user, account.getUsername(), account.getId());
                    dto = BeanUtil.toBean(account, AuthorizeRespDTO.class);
                    dto.setAccess_token(newAccessToken);
                    dto.setRefresh_token(newRefreshJwt);
                    dto.setAccess_expire(utils.expireTime());
                    dto.setRefresh_expire(utils.reExpireTime());
                }
            }
        }
        return dto;
    }
```

