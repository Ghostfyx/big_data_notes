# System源码分析

## 1. getEnv

获取当前JVM所在系统的环境变量，返回的变量基本都是跟系统相关

```java
/**
     * Returns an unmodifiable string map view of the current system environment.
     * 返回当前系统环境的不可修改的String类型Map集合
     * The environment is a system-dependent mapping from names to
     * values which is passed from parent to child processes.
     * 环境是从名称-值的系统依赖的映射，从父进程传递到子进程。
     *
     * <p>If the system does not support environment variables, an
     * empty map is returned.
     *
     * <p>The returned map will never contain null keys or values.
     * Attempting to query the presence of a null key or value will
     * throw a {@link NullPointerException}.  Attempting to query
     * the presence of a key or value which is not of type
     * {@link String} will throw a {@link ClassCastException}.
     *
     * <p>The returned map and its collection views may not obey the
     * general contract of the {@link Object#equals} and
     * {@link Object#hashCode} methods.
     *
     * <p>The returned map is typically case-sensitive on all platforms.
     *
     * <p>If a security manager exists, its
     * {@link SecurityManager#checkPermission checkPermission}
     * method is called with a
     * <code>{@link RuntimePermission}("getenv.*")</code>
     * permission.  This may result in a {@link SecurityException} being
     * thrown.
     *
     * <p>When passing information to a Java subprocess,
     * <a href=#EnvironmentVSSystemProperties>system properties</a>
     * are generally preferred over environment variables.
     *
     * @return the environment as a map of variable names to values
     * @throws SecurityException
     *         if a security manager exists and its
     *         {@link SecurityManager#checkPermission checkPermission}
     *         method doesn't allow access to the process environment
     * @see    #getenv(String)
     * @see    ProcessBuilder#environment()
     * @since  1.5
     */
    public static java.util.Map<String,String> getenv() {
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("getenv.*"));
        }

        return ProcessEnvironment.getenv();
    }
```

```java
/**
     * Gets the value of the specified environment variable. An
     * environment variable is a system-dependent external named
     * value.
     *
     * <p>If a security manager exists, its
     * {@link SecurityManager#checkPermission checkPermission}
     * method is called with a
     * <code>{@link RuntimePermission}("getenv."+name)</code>
     * permission.  This may result in a {@link SecurityException}
     * being thrown.  If no exception is thrown the value of the
     * variable <code>name</code> is returned.
     *
     * <p><a name="EnvironmentVSSystemProperties"><i>System
     * properties</i> and <i>environment variables</i></a> are both
     * conceptually mappings between names and values.  Both
     * mechanisms can be used to pass user-defined information to a
     * Java process.  Environment variables have a more global effect,
     * because they are visible to all descendants of the process
     * which defines them, not just the immediate Java subprocess.
     * They can have subtly different semantics, such as case
     * insensitivity, on different operating systems.  For these
     * reasons, environment variables are more likely to have
     * unintended side effects.  It is best to use system properties
     * where possible.  Environment variables should be used when a
     * global effect is desired, or when an external system interface
     * requires an environment variable (such as <code>PATH</code>).
     *
     * <p>On UNIX systems the alphabetic case of <code>name</code> is
     * typically significant, while on Microsoft Windows systems it is
     * typically not.  For example, the expression
     * <code>System.getenv("FOO").equals(System.getenv("foo"))</code>
     * is likely to be true on Microsoft Windows.
     *
     * @param  name the name of the environment variable
     * @return the string value of the variable, or <code>null</code>
     *         if the variable is not defined in the system environment
     * @throws NullPointerException if <code>name</code> is <code>null</code>
     * @throws SecurityException
     *         if a security manager exists and its
     *         {@link SecurityManager#checkPermission checkPermission}
     *         method doesn't allow access to the environment variable
     *         <code>name</code>
     * @see    #getenv()
     * @see    ProcessBuilder#environment()
     */
    public static String getenv(String name) {
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("getenv."+name));
        }

        return ProcessEnvironment.getenv(name);
    }
```

## 2. getProperty

getProperty方法返回的变量基本都是跟java程序有关

| 序号 | 键                            | 相关值的描述                      |
| ---- | ----------------------------- | --------------------------------- |
| 1    | java.version                  | Java 运行时环境版本               |
| 2    | java.vendor                   | Java 运行时环境供应商             |
| 3    | java.vendor.url               | Java 供应商的 URL                 |
| 4    | java.home                     | Java 安装目录                     |
| 5    | java.vm.specification.version | Java 虚拟机规范版本               |
| 6    | java.vm.specification.vendor  | Java 虚拟机规范供应商             |
| 7    | java.vm.specification.name    | Java 虚拟机规范名称               |
| 8    | java.vm.version               | Java 虚拟机实现版本               |
| 9    | java.vm.vendor                | Java 虚拟机实现供应商             |
| 10   | java.vm.name                  | Java 虚拟机实现名称               |
| 11   | java.specification.version    | Java 运行时环境规范版本           |
| 12   | java.specification.vendor     | Java 运行时环境规范供应商         |
| 13   | java.specification.name       | Java 运行时环境规范名称           |
| 14   | java.class.version            | Java 类格式版本号                 |
| 15   | java.class.path               | Java 类路径                       |
| 16   | java.library.path             | 加载库时搜索的路径列表            |
| 17   | java.io.tmpdir                | 默认的临时文件路径                |
| 18   | java.compiler                 | 要使用的 JIT 编译器的名称         |
| 19   | java.ext.dirs                 | 一个或多个扩展目录的路径          |
| 20   | os.name                       | 操作系统的名称                    |
| 21   | os.arch                       | 操作系统的架构                    |
| 22   | os.version                    | 操作系统的版本                    |
| 23   | file.separator                | 文件分隔符（在 UNIX 系统中是“/”） |
| 24   | path.separator                | 路径分隔符（在 UNIX 系统中是“:”） |
| 25   | line.separator                | 行分隔符（在 UNIX 系统中是“/n”）  |
| 26   | user.name                     | 用户的账户名称                    |
| 27   | user.home                     | 用户的主目录                      |
| 28   | user.dir                      | 用户的当前工作目录                |

```java
 * <p>
     * Multiple paths in a system property value are separated by the path
     * separator character of the platform.
     * <p>
     * Note that even if the security manager does not permit the
     * <code>getProperties</code> operation, it may choose to permit the
     * {@link #getProperty(String)} operation.
     *
     * @return     the system properties
     * @exception  SecurityException  if a security manager exists and its
     *             <code>checkPropertiesAccess</code> method doesn't allow access
     *              to the system properties.
     * @see        #setProperties
     * @see        java.lang.SecurityException
     * @see        java.lang.SecurityManager#checkPropertiesAccess()
     * @see        java.util.Properties
     */
    public static Properties getProperties() {
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPropertiesAccess();
        }

        return props;
    }
```

```java
/**
     * Gets the system property indicated by the specified key.
     * <p>
     * First, if there is a security manager, its
     * <code>checkPropertyAccess</code> method is called with the key as
     * its argument. This may result in a SecurityException.
     * <p>
     * If there is no current set of system properties, a set of system
     * properties is first created and initialized in the same manner as
     * for the <code>getProperties</code> method.
     *
     * @param      key   the name of the system property.
     * @return     the string value of the system property,
     *             or <code>null</code> if there is no property with that key.
     *
     * @exception  SecurityException  if a security manager exists and its
     *             <code>checkPropertyAccess</code> method doesn't allow
     *              access to the specified system property.
     * @exception  NullPointerException if <code>key</code> is
     *             <code>null</code>.
     * @exception  IllegalArgumentException if <code>key</code> is empty.
     * @see        #setProperty
     * @see        java.lang.SecurityException
     * @see        java.lang.SecurityManager#checkPropertyAccess(java.lang.String)
     * @see        java.lang.System#getProperties()
     */
    public static String getProperty(String key) {
        checkKey(key);
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPropertyAccess(key);
        }

        return props.getProperty(key);
    }
```

```java
/**
     * Gets the system property indicated by the specified key.
     * <p>
     * First, if there is a security manager, its
     * <code>checkPropertyAccess</code> method is called with the
     * <code>key</code> as its argument.
     * <p>
     * If there is no current set of system properties, a set of system
     * properties is first created and initialized in the same manner as
     * for the <code>getProperties</code> method.
     *
     * @param      key   the name of the system property.
     * @param      def   a default value.
     * @return     the string value of the system property,
     *             or the default value if there is no property with that key.
     *
     * @exception  SecurityException  if a security manager exists and its
     *             <code>checkPropertyAccess</code> method doesn't allow
     *             access to the specified system property.
     * @exception  NullPointerException if <code>key</code> is
     *             <code>null</code>.
     * @exception  IllegalArgumentException if <code>key</code> is empty.
     * @see        #setProperty
     * @see        java.lang.SecurityManager#checkPropertyAccess(java.lang.String)
     * @see        java.lang.System#getProperties()
     */
    public static String getProperty(String key, String def) {
        checkKey(key);
        SecurityManager sm = getSecurityManager();
        if (sm != null) {
            sm.checkPropertyAccess(key);
        }

        return props.getProperty(key, def);
    }
```

