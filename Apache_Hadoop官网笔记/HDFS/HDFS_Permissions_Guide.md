# HDFS Permissions Guide

 ## 1. 概述

Hadoop分布式文件系统(HDFS)为共享大部分POSIX模型的文件和目录实现权限模型。每个文件和目录都与一个所有者和一个组相关联。文件与目录对于所有者的用户，该组成员的其他用户以及所有其他组的用户分别具有单独的权限。 对于文件，需要r权限才能读取文件，而w权限才需要写入或附加到文件。 对于目录，需要r权限才能列出目录的内容，需要w权限才能创建或删除文件或目录，并且需要x权限才能访问目录的子级。

HDFS文件模型与POSIX模型相比，文件没有setuid或setgid位，因为没有可执行文件的概念。对于目录，没有setuid或setgid bits目录来简化。可以对目录设置特殊权限粘滞位，以防止除超级用户，目录所有者或文件所有者以外的任何人删除或移动目录中的文件。总而言之，文件或目录的权限就是其模式。通常使用Unix习惯表示和显示模式。创建文件或目录时，其所有者是客户端进程的用户身份，其组是父目录的组(BSD规则)。

HDFS还提供了对POSIX ACL(Access Control Lists，访问控制列表)的可选支持，以使用针对特定命名用户或命名组的更细粒度的规则来扩展文件权限。 本文档后面将详细讨论ACL。

每个访问HDFS的客户端进程都有一个由用户名和组列表两部分组成的身份。在客户端进程每次访问HDFS中的文件或目录时，都需要进行权限校验：

- 如果客户端进程的用户名与文件/目录所有者匹配，则将测试所有者权限；
- 否则，如果客户端进程的组与文件/目录组列表匹配，则将测试组权限；
- 否则，测试其他权限

如果权限校验失败，则客户端操作失败。

## 2. 用户身份

从Hadoop 0.22开始，Hadoop支持两种操作模式来确定用户的身份，由`hadoop.security.authentication`属性指定：

- **Simple**

	在这种操作模式下，客户端进程的身份由其所在主机操作系统确定。 在类似Unix的系统上，用户名等效于“ who am i”。

- **kerberos**

	在Kerberized操作中，客户端进程的身份由其Kerberos凭据确定。 例如，在以Kerberized环境中，用户可以使用kinit实用程序获取Kerberos票据授予票据(Ticket-GrantingTicket，TGT)，并使用klist确定其当前主体。将Kerberos主体映射到HDFS用户名时，除用户名外的所有部分均被删除。 例如，委托人`todd/foobar@CORP.COMPANY.COM`将充当HDFS上的简单用户名todd。

无论操作模式如何，用户标识机制都是HDFS本身固有的。 HDFS内没有用于创建用户身份，建立组或处理用户凭据的规定。

## 3. 用户组映射

如上所述确定用户名后，组列表将由`hadoop.security.group.mapping`属性配置的组映射服务确定。 有关详细信息，请参见 [Hadoop Groups Mapping](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/GroupsMapping.html)。

## 4. 权限校验

每个HDFS操作都要求用户具有特定的权限(READ，WRITE和EXECUTE的某种组合)，通过文件所有权，组成员身份或其他权限授予。 一个操作可能会在路径的多个部分上执行权限检查，而不仅仅是最终部分。 而且，某些操作取决于对路径所有者的检查。

所有操作都需要遍历访问。 遍历访问需要对路径的所有现有部分(最终路径除外)具有EXECUTE权限。例如，对于任何访问/foo/bar/baz的操作，调用者必须对/，/foo和/foo/bar具有EXECUTE权限。

下表描述了HDFS对路径的每个部分执行的权限检查：

- **Ownership**：是否检查调用方是否是路径的所有者。通常，更改文件/目录的所有权或权限元数据的操作要求调用者是所有者。
- **Parent**：请求路径的父目录。 例如，对于路径/foo/bar/baz，父路径为/foo/bar。
- **Ancestor**：请求路径的最后一个存在父路径。例如，对于路径/foo/bar/baz，如果存在/foo/bar，则祖先路径为/foo/bar。 如果/foo存在但/foo/bar不存在，则祖先路径为/foo。
- **Final**：请求路径的最终组成部分。 例如，对于路径/foo/bar/baz，最终路径组件是/foo/bar/baz。
- **Sub-tree**：对于作为目录的路径，该目录本身及其所有子目录都是递归的。 例如，对于路径/foo/bar/baz(具有2个名为buz和boo的子目录)。子树为：/foo/bar/baz, /foo/bar/baz/buz 与/foo/bar/baz/boo。

| Operation             | Ownership | Parent          | Ancestor            | Final                              | Sub-tree             |
| :-------------------- | :-------- | :-------------- | :------------------ | :--------------------------------- | :------------------- |
| append                | NO        | N/A             | N/A                 | WRITE                              | N/A                  |
| concat                | NO [2]    | WRITE (sources) | N/A                 | READ(sources), WRITE (destination) | N/A                  |
| create                | NO        | N/A             | WRITE               | WRITE [1]                          | N/A                  |
| createSnapshot        | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| delete                | NO [2]    | WRITE           | N/A                 | N/A                                | READ, WRITE, EXECUTE |
| deleteSnapshot        | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| getAclStatus          | NO        | N/A             | N/A                 | N/A                                | N/A                  |
| getBlockLocations     | NO        | N/A             | N/A                 | READ                               | N/A                  |
| getContentSummary     | NO        | N/A             | N/A                 | N/A                                | READ, EXECUTE        |
| getFileInfo           | NO        | N/A             | N/A                 | N/A                                | N/A                  |
| getFileLinkInfo       | NO        | N/A             | N/A                 | N/A                                | N/A                  |
| getLinkTarget         | NO        | N/A             | N/A                 | N/A                                | N/A                  |
| getListing            | NO        | N/A             | N/A                 | READ, EXECUTE                      | N/A                  |
| getSnapshotDiffReport | NO        | N/A             | N/A                 | READ                               | READ                 |
| getStoragePolicy      | NO        | N/A             | N/A                 | READ                               | N/A                  |
| getXAttrs             | NO        | N/A             | N/A                 | READ                               | N/A                  |
| listXAttrs            | NO        | EXECUTE         | N/A                 | N/A                                | N/A                  |
| mkdirs                | NO        | N/A             | WRITE               | N/A                                | N/A                  |
| modifyAclEntries      | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| removeAcl             | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| removeAclEntries      | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| removeDefaultAcl      | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| removeXAttr           | NO [2]    | N/A             | N/A                 | WRITE                              | N/A                  |
| rename                | NO [2]    | WRITE (source)  | WRITE (destination) | N/A                                | N/A                  |
| renameSnapshot        | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| setAcl                | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| setOwner              | YES [3]   | N/A             | N/A                 | N/A                                | N/A                  |
| setPermission         | YES       | N/A             | N/A                 | N/A                                | N/A                  |
| setReplication        | NO        | N/A             | N/A                 | WRITE                              | N/A                  |
| setStoragePolicy      | NO        | N/A             | N/A                 | WRITE                              | N/A                  |
| setTimes              | NO        | N/A             | N/A                 | WRITE                              | N/A                  |
| setXAttr              | NO [2]    | N/A             | N/A                 | WRITE                              | N/A                  |
| truncate              | NO        | N/A             | N/A                 | WRITE                              | N/A                  |

[1]表示仅在调用使用覆盖选项并且路径上存在现有文件时，才需要在创建过程中对最终路径组件进行`WRITE`访问。

[2]表示如果设置了`sticky`位，则检查父目录上的`WRITE`权限的任何操作也会检查所有权。

[3]表示调用`setOwner`更改拥有文件的用户需要HDFS超级用户访问权限。 更改组不需要HDFS超级用户访问权限，但是调用者必须是文件的所有者和指定组的成员。

## 5. 理解权限校验实现

每个文件或目录操作将完整路径名传递给NameNode，并且权限校验为每个操作沿文件路径检查。客户端框架将隐式地将用户身份与与NameNode的连接关联起来，从而减少了对现有客户端API进行更改。一直存在这样的情况：对文件执行一项操作成功时，重复操作可能会失败，因为该文件或路径上的某些目录已不存在。例如，当客户端第一次开始读取文件时，它会向NameNode发出第一个请求，以发现文件的第一个块的位置。查找其他块的第二个请求可能会失败。另一方面，删除文件不会取消已经知道文件块位置的客户端的访问。随着权限的增加，在两次请求之间可能会撤消客户端对文件的访问。同样，更改权限不会撤消已经知道文件阻止内容的客户端的访问。

## 6. 对文件系统API的更改

如果权限检查失败，则所有使用path参数的方法都将引发AccessControlException。

新的方法：

- `public FSDataOutputStream create(Path f, FsPermission permission, boolean overwrite, int bufferSize, short replication, long blockSize, Progressable progress) throws IOException;`
- `public boolean mkdirs(Path f, FsPermission permission) throws IOException;`
- `public void setPermission(Path p, FsPermission permission) throws IOException;`
- `public void setOwner(Path p, String username, String groupname) throws IOException;`
- `public FileStatus getFileStatus(Path f) throws IOException;`还将返回与该路径关联的用户，组和模式。

新文件或目录的模式被设置为配置参数的umask限制。当使用之前的create(path, …) 方法(不带权限参数)，新文件的模式为` 0666 & ^umask`。当使用新的 create(path, permission, …)方法(具有权限参数p)，新文件的模式为`P & ^umask & 0666`。使用现有的`mkdirs(path)`方法创建新目录时 (没有permission参数)，新目录的模式为`0777＆^ umask`。 当使用新的`mkdirs(path, Permission)`方法(具有权限参数P)时，新目录的模式为`P＆^ umask＆0777`。

## 7. 对应用Shell命令的更改

新操作：

- `chmod [-R] mode file ...`

	仅文件的所有者或超级用户被允许更改文件的模式。

- `chgrp [-R] group file ...`

	调用chgrp的用户必须属于指定的组，并且是文件的所有者，或者是超级用户。

- ```
	chown [-R] [owner][:[group]] file ...
	```

	文件的所有者只能由超级用户更改。

- `ls file ...`

- `lsr file ...`

重新格式化输出以显示所有者，组和模式。

## 8. 超级用户

超级用户是与NameNode进程本身具有相同标识的用户。简单来说，如果启动了NameNode的用户，即为超级用户。超级用户可以执行任何操作，因为超级用户的权限检查永远不会失败。 谁是超级用户并没有持久的观念。 当NameNode启动时，进程标识将确定谁是当前的超级用户。HDFS超级用户不必是NameNode主机的超级用户，也不必所有群集都具有相同的超级用户。 此外，在个人工作站上运行HDFS的实验人员可以方便地成为本地的超级用户，而无需进行任何配置。

另外，管理员可以使用配置参数来识别专有组。 如果设置，则该组的成员也是超级用户。

## 9. web服务

默认情况下，Web服务器的身份是配置参数。 也就是说，NameNode没有真实用户身份的概念，但是Web服务器的行为就像它具有管理员选择的用户的身份(用户和组)一样。 除非所选择的身份与超级用户匹配，否则Web服务器可能无法访问部分名称空间。

## 10. 访问控制列表ACLs

除了传统的POSIX权限模型外，HDFS还支持POSIX ACL(访问控制列表)。 ACL对于实现不同于用户和组的组织层次结构权限很有用。 ACL提供了一种方法，可以为特定的命名用户或命名组设置不同的权限，而不仅仅是文件的所有者和文件的组。

默认情况下，禁用对ACL的支持，并且NameNode禁止创建ACL。 要启用ACL，则NameNode配置中将dfs.namenode.acls.enabled设置为true。

ACL由一组ACL条目组成。 每个ACL条目都为一个特定的用户或组命名，用于授予或拒绝对该特定用户或组的读取，写入和执行权限。 例如：

```
user::rw-
user:bruce:rwx                  #effective:r--
group::r-x                      #effective:r--
group:sales:rwx                 #effective:r--
mask::r--
other::r--
```

ACL条目由类型，可选名称和权限字符串组成，使用":"作为每个字段之间的分隔符。文件所有者具有读/写权限，文件组具有读取/执行权限，其他人具有读取权限。到目前为止，这等效于将文件的权限位设置为654。

此外，还为命名用户bruce和命名组sales提供了2个扩展ACL条目，它们均被授予完全访问权限。掩码是一个特殊的ACL条目，它过滤授予所有命名用户条目和命名组条目以及未命名组条目的权限。在示例中，掩码仅具有读取权限，并且可以看到几个ACL条目的有效权限已被相应地过滤。

每个ACL必须有一个掩码。如果用户在设置ACL时未提供掩码，则会通过计算将被掩码过滤的所有条目的权限并集来自动插入掩码。

在具有ACL的文件上运行chmod实际上会更改掩码的权限。由于掩码充当过滤器，因此有效地限制了所有扩展ACL条目的权限，而不是仅更改组条目并可能丢失其他扩展ACL条目。

模型还区分了“访问ACL”和“默认ACL”，“访问ACL”定义了权限检查期间要执行的规则，“默认ACL”定义了新的子文件或子目录在创建过程中自动接收的ACL条目。例如：

```
user::rwx
group::r-x
other::r-x
default:user::rwx
default:user:bruce:rwx          #effective:r-x
default:group::r-x
default:group:sales:rwx         #effective:r-x
default:mask::r-x
default:other::r-x
```

仅目录可以具有默认ACL。创建新文件或子目录后，它将自动将其父级的默认ACL复制到其自己的访问ACL中。新的子目录还将其复制到其自己的默认ACL。这样，当创建新的子目录时，将通过文件系统树的任意深层次向下复制默认ACL。

新子级访问ACL中的确切权限值需要通过mode参数进行过滤。考虑到默认umask 022，对于新目录，这通常是755，对于新文件，通常是644。 mode参数过滤未命名用户(文件所有者)，掩码和其他用户的复制的权限值。使用此特定示例ACL，并为模式创建带有755的新子目录，此模式过滤对最终结果没有影响。但是，如果我们考虑使用模式644创建文件，则模式过滤会导致新文件的ACL对未命名用户(文件所有者)进行读写操作，对掩码进行读取，对其他用户进行读取。此掩码还意味着仅读取对命名用户bruce和命名组sales的有效权限。

请注意，复制是在创建新文件或子目录时进行的。对父级默认ACL的后续更改不会更改现有子级。

默认ACL必须具有所有必需的最小ACL条目，包括未命名的用户（文件所有者），未命名的组（文件组）和其他条目。如果用户在设置默认ACL时未提供这些条目之一，则将通过复制访问ACL中的相应权限自动插入条目，如果没有访问ACL，则复制权限位。默认ACL还必须具有掩码。如上所述，如果未指定掩码，则通过计算将被掩码过滤的所有条目的权限并集，自动插入掩码。

考虑具有ACL的文件时，权限检查算法更改为：

- 如果用户名与文件的所有者匹配，则将测试所有者权限；
- 否则，如果用户名与命名的用户条目之一中的名称匹配，则将测试这些许可权，并通过掩码许可权进行过滤；
- 否则，如果文件组与组列表中的任何成员匹配，并且如果由掩码过滤的这些权限授予访问权限，则使用这些权限。
- 否则，如果存在与组列表成员匹配的命名组条目，并且如果这些权限由掩码授予访问权限过滤，则使用这些权限；
- 否则，如果文件组或任何命名的组条目与组列表的成员匹配，但是任何这些权限均未授予访问权限，则访问被拒绝；
- 否则，将测试文件的其他权限。

最佳实践是依靠传统的权限位来实现大多数权限要求，并定义较少数量的ACL以通过一些例外规则来扩展权限位。与仅具有权限位的文件相比，具有ACL的文件会在NameNode中的内存中产生额外的开销。

## 11. ACL文件系统API

新方法：

- `public void modifyAclEntries(Path path, List aclSpec) throws IOException;`
- `public void removeAclEntries(Path path, List aclSpec) throws IOException;`
- `public void public void removeDefaultAcl(Path path) throws IOException;`
- `public void removeAcl(Path path) throws IOException;`
- `public void setAcl(Path path, List aclSpec) throws IOException;`
- `public AclStatus getAclStatus(Path path) throws IOException;`

## 12. ACL命令行

- `hdfs dfs -getfacl [-R] <path>`

	显示文件和目录的访问控制列表。 如果目录具有默认ACL，则getfacl还将显示默认ACL
	
- `hdfs dfs -setfacl [-R] [-b |-k -m |-x  ] |[--set  ]`

	设置文件和目录的访问控制列表

- `hdfs dfs -ls <args>`

	ls的输出将在带有ACL的任何文件或目录的权限字符串后附加一个“ +”字符

## 13. 配置参数

- `dfs.permissions.enabled = true`

	如果是，请按照此处所述使用权限系统。 如果否，则关闭权限检查，但其他所有行为均保持不变。 从一个参数值切换到另一个参数值不会更改模式，所有者或文件或目录组。 无论权限是打开还是关闭，chmod，chgrp，chown和setfacl都会始终检查权限。 这些功能仅在权限上下文中有用，因此不存在向后兼容性问题。 此外，这允许管理员在打开常规权限检查之前可靠地设置所有者和权限。

- `dfs.web.ugi = webuser,webgroup`

	Web服务器要使用的用户名。 将此设置为超级用户的名称将允许任何Web客户端查看所有内容。 将其更改为其他方式未使用的身份，可使Web客户端仅查看使用“其他”权限可见的内容。 可以将其他组添加到以逗号分隔的列表中。

- `dfs.permissions.superusergroup = supergroup`

	超级用户组的名称

- `fs.permissions.umask-mode = 0022`

	创建文件和目录时使用的umask。 对于配置文件，可以使用十进制值18。

- `dfs.cluster.administrators = ACL-for-admins`

	指定为ACL的群集的管理员。 这控制了谁可以访问HDFS中的默认servlet等

- `dfs.namenode.acls.enabled = true`

	设置为true以启用对HDFS ACL的支持。 默认情况下，禁用ACL。 禁用ACL时，NameNode拒绝所有设置ACL的尝试。

- `dfs.namenode.posix.acl.inheritance.enabled`

	设置为true以启用POSIX样式ACL继承。 默认启用。 启用该选项后，当创建请求来自兼容的客户端时，NameNode会将父目录中的默认ACL应用于创建模式，并忽略客户端umask。 如果未找到默认ACL，它将应用客户端umask。