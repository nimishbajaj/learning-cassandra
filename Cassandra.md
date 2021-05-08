# Cassandra

# Introduction

## Replication: ACID is a lie

Data is replicated asynchronously, this is known as replication lag. When the client decides to write to the master it takes some time to replicate to the salve. If the client decided to do a read, it gets the old data back. Hence the data is not consistent.

![image-20210508185719983](/home/nimish/.config/Typora/typora-user-images/image-20210508185719983.png)

![image-20210508185756733](/home/nimish/.config/Typora/typora-user-images/image-20210508185756733.png)