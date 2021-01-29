# Cache

The internal cache strategy of **Hector ORM** is to keep in cache system:

- Schema structure
- Data type associations
- Some entities reflection data

The cache system of **Hector ORM** is based on **psr/simple-cache** specifications with internal abstraction.

The factory **Hector/Orm/OrmFactory** helps you to keep and save data in cache system.