---
title: 学习Spring Boot：（十四）spring-shiro的密码加密
date: 2018-02-23 10:08:15
tags: [Spring Boot,shiro]
categories: 学习笔记
[1075199251,张凯,18772383543,wuwii,追光者w,有梦想的咸鱼,一棵树站在原野上,Slience]
---

### 前言
前面配置了怎么使用 shiro ，这次研究下怎么使用spring shiro的密码加密，并且需要在新增、更新用户的时候，实现生成盐，加密后的密码进行入库操作。

<!--more-->

### 正文
#### 配置凭证匹配器
```java
    @Bean
    public HashedCredentialsMatcher hashedCredentialsMatcher() {
        HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
        hashedCredentialsMatcher.setHashAlgorithmName("SHA-256");//散列算法:MD2、MD5、SHA-1、SHA-256、SHA-384、SHA-512等。
        hashedCredentialsMatcher.setHashIterations(1);//散列的次数，默认1次， 设置两次相当于 md5(md5(""));
        return hashedCredentialsMatcher;
    }

    /**
     * 注册身份验证
     * @param hashedCredentialsMatcher 凭证匹配器
     * @return
     */
    @Bean
    public OAuth2Realm oAuth2Realm(HashedCredentialsMatcher hashedCredentialsMatcher) {
        OAuth2Realm oAuth2Realm = new OAuth2Realm();
        oAuth2Realm.setCredentialsMatcher(hashedCredentialsMatcher);
        return oAuth2Realm;
    }
```
这样就把凭证匹配器注册到身份验证的 Realm 中，在用户进行登陆操作的时候，在 Realm 中的 `doGetAuthenticationInfo` 方法中使用这种方法进行用户身份认证：
```java
return new SimpleAuthenticationInfo(
                user, // 存入凭证的信息，登陆成功后可以使用 SecurityUtils.getSubject().getPrincipal();在任何地方使用它
                user.getPassword(),
                ByteSource.Util.bytes(user.getSalt()), // 加盐，
                getName());
```
#### 生成加密密码
```java
    /**
     * 随机生成 salt 需要指定 它的字符串的长度
     *
     * @param len 字符串的长度
     * @return salt
     */
    public static String generateSalt(int len) {
        //一个Byte占两个字节
        int byteLen = len >> 1;
        SecureRandomNumberGenerator secureRandom = new SecureRandomNumberGenerator();
        return secureRandom.nextBytes(byteLen).toHex();
    }

    /**
     * 获取加密后的密码，使用默认hash迭代的次数 1 次
     *
     * @param hashAlgorithm hash算法名称 MD2、MD5、SHA-1、SHA-256、SHA-384、SHA-512、etc。
     * @param password      需要加密的密码
     * @param salt          盐
     * @return 加密后的密码
     */
    public static String encryptPassword(String hashAlgorithm, String password, String salt) {
        return encryptPassword(hashAlgorithm, password, salt, 1);
    }

    /**
     * 获取加密后的密码，需要指定 hash迭代的次数
     *
     * @param hashAlgorithm  hash算法名称 MD2、MD5、SHA-1、SHA-256、SHA-384、SHA-512、etc。
     * @param password       需要加密的密码
     * @param salt           盐
     * @param hashIterations hash迭代的次数
     * @return 加密后的密码
     */
    public static String encryptPassword(String hashAlgorithm, String password, String salt, int hashIterations) {
        SimpleHash hash = new SimpleHash(hashAlgorithm, password, salt, hashIterations);
        return hash.toString();
    }
```
然后将生成出来的盐，加密密码插入到数据库就完成了。
```java
    @Override
    public void save(SysUserEntity sysUser) {
        sysUser.setCreateDate(new Date());
        // 密码加密 方式很多，任选
       /* String salt = RandomStringUtils.randomAlphanumeric(20);
        sysUser.setPassword(new Sha256Hash(sysUser.getPassword(), salt).toHex());*/

        String salt = ShiroUtils.generateSalt(20);
        sysUser.setPassword(ShiroUtils.encryptPassword("SHA-256", sysUser.getPassword(), salt));
        sysUser.setSalt(salt);
        sysUser.setUsername(sysUser.getEmail());
        sysUser.setStatus(SysConstant.SysUserStatus.ACTIVE);
        sysUser.setType(SysConstant.SysUserType.USER);
        sysUserDao.save(sysUser);
    }
```
