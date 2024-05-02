JdbcRealm中创建用户的一般写法：

```java
public String register(User user) {
        RandomNumberGenerator gen = new SecureRandomNumberGenerator();
        ByteSource salt = gen.nextBytes();

        String hashedPasswordBase64 = new Sha256Hash(user.getPassword(), salt, 1024).toBase64();

        user.setPassword(hashedPasswordBase64);
        user.setSalt(salt.getBytes());

        try {
            userDAO.create(user);
        } catch (SQLException e) {
            e.printStackTrace();
            //TOOD:log error
        }
        return "redirect:login";
    }
```

这个里面一个比较麻烦的问题在于salt该如何存放。nextBytes()方法返回的实际值是SimpleByteSource，有些地方直接使用salt.toString()存作字符串，SimpleByteSource重写了toString方法，实际上返回的是base64编码。问题在于Jdbc并不能使用这种方式存放的数据。

看下JdbcRealm的doGetAuthenticationInfo方法：

```java
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

        UsernamePasswordToken upToken = (UsernamePasswordToken) token;
        String username = upToken.getUsername(); // Null username is invalid
        if (username == null) { throw new AccountException("Null usernames are not allowed by this realm.");
        }

        Connection conn = null;
        SimpleAuthenticationInfo info = null; try {
            conn = dataSource.getConnection();

            String password = null;
            String salt = null; switch (saltStyle) { case NO_SALT:
                password = getPasswordForUser(conn, username)\[0\]; break; case CRYPT: // TODO: separate password and hash from getPasswordForUser\[0\]
                throw new ConfigurationException("Not implemented yet"); //break;
            case COLUMN:
                String\[\] queryResults = getPasswordForUser(conn, username);
                password = queryResults\[0\];
                salt = queryResults\[1\]; break; case EXTERNAL:
                password = getPasswordForUser(conn, username)\[0\];
                salt = getSaltForUser(username);
            } if (password == null) { throw new UnknownAccountException("No account found for user \[" + username + "\]");
            }

            info = new SimpleAuthenticationInfo(username, password.toCharArray(), getName()); if (salt != null) {
                info.setCredentialsSalt(ByteSource.Util.bytes(salt));
            }

        } catch (SQLException e) { final String message = "There was a SQL error while authenticating user \[" + username + "\]"; if (log.isErrorEnabled()) {
                log.error(message, e);
            } // Rethrow any SQL errors as an authentication exception
            throw new AuthenticationException(message, e);
        } finally {
            JdbcUtils.closeConnection(conn);
        } return info;
    }
```

可以看出JdbcRealm是吧密码和盐都当作字符串读取，而ByteSource本质上存放的是byte数组。深究下去，实际上是使用UTF-8解码字符串得到了byte\[\]。但问题在于，String并不是什么byte都可以存放的，不论是JVM内部使用的UTF-16，还是这里编解码使用的UTF-8，都是有一定格式要求的。不是什么byte\[\]都可以放进去然后原封不动的拿出来，更何况中间还经过了一次MySQL。

更重要的是，根本没有必要使用字符串存放，ByteSource.Util.bytes本身就有byte\[\]的重载，而base64也谈不上什么加密。直接存放二进制是不存在问题的。实现起来也很简单。重写doGetAuthenticationInfo，使用byte\[\]创建SimpleByteSource即可。