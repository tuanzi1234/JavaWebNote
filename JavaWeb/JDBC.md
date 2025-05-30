## 一、JDBC
* **JDBC：** Java Database Connectivity，Java语言操作关系型数据库的一套API。
* **使用JDBC操作数据库执行修改语句步骤：**
```java
 /*
    * JDBC操作
    * */
    @Test
    public void test() throws Exception {
        //1.注册驱动
        Class.forName("com.mysql.cj.jdbc.Driver");
        //2.获取数据库连接
        Connection  connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/canteen_manage", "root", "lvbohan2003");
        //3.获取数据库操作对象
        Statement statement = connection.createStatement();
        //4.执行SQL语句
        int count = statement.executeUpdate( "update admin set name = 'admin020' where id = 5");
        System.out.println("sql语句执行完毕影响的记录数" + count);
        //5.释放资源
        statement.close();
        connection.close();
    }
```
* **使用JDBC操作数据库执行查询语句步骤：**
```java
 //执行DQL语句
    @Test
    public void test1(){
        //1.初始化resultSet、preparedStatement、connection
        ResultSet resultSet = null;
        PreparedStatement preparedStatement = null;
        Connection connection = null;
        try {
            //2.注册驱动
            Class.forName("com.mysql.cj.jdbc.Driver");
            //3.获取数据库连接
            connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/canteen_manage", "root", "lvbohan2003");
            //4.执行SQL语句
            String sql = "select id, username, password, name, avatar, phone, email from admin where id = ?";//预编译SQL语句，？为占位符
            //5.传入id的值
            preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setInt(1, 5);//1为占位符的索引(从1开始计数)，5为传入的值
            //6.解析结果，将结果封装进Admin中
            resultSet = preparedStatement.executeQuery();
            while (resultSet.next()) {
                Admin admin = new Admin();
                admin.setId(resultSet.getInt("id"));
                admin.setUsername(resultSet.getString("username"));
                admin.setPassword(resultSet.getString("password"));
                admin.setName(resultSet.getString("name"));
                admin.setAvatar(resultSet.getString("avatar"));
                admin.setPhone(resultSet.getString("phone"));
                admin.setEmail(resultSet.getString("email"));
                System.out.println(admin);
            }
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            try {
                //7.释放资源，放入finally中，确保即使出现异常，资源也能被释放
                resultSet.close();
                preparedStatement.close();
                connection.close();
                System.out.println("sql语句执行完毕");
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
    }
```
## 二、关于预编译SQL
* **预编译SQL优势一：** 可以防止SQL注入，更安全。
  * SQL注入：通过控制输入来修改实现定义的SQL语句，对服务器进行攻击。
* **预编译SQL优势二：** 提升性能。