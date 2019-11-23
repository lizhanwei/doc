
# Create

## 创建Node
```java
create (n:Person{name:'li',title:'test'})
create (n:Person{name:'zhang',title:'haha'})
```

## 创建关系
```java
match (a:Person),(b:Person)
where a.name='li' and b.name='zhang'
create (a)-[r:reltype]->(b)
return type(r)
```

## 创建Node同时创建关系
```java
create (:Person{name:'wang',title:'test'})-[:hasfriend]->(:Person{name:'zhao',title:'test'})

match (a:Person),(b:Person)
where a.name='wang' and b.name='li'
create (a)-[:hasfriend]->(b)
```

# Match
## 按照属性值查
```java
查节点
match (n:Person)
where n.name='li'
return n.title

select title from Person where name='li'
```

## 按照关系名称查
```java
match (hasfriend{name:'wang'})--(n:Person)
return n

//王有哪些朋友
```

## 查关系
```java
match (:Person{name:'wang'})-[r]->(:Person{name:'li'})
return type(r)
```

# With
## 排序
```java
match (n:Person)
with n
order by n.name desc limit 2
return n
```

# where
```java
match (n:Person)
where not n.name ='zhao' and n.title='test'
return n

//and、or、not、exists、STARTS WITH、ENDS WITH、=~ 'regexp'、


```

# Order
```java
match (n:Person)
return n
order by n.name
skip 1
limit 2
```
# delete
## 删除Node
```java

```

## 删除关系
```java
match (:Person{name:'zhang'})<-[r]-()
delete r

```

## 删除节点
```java
match (n:Person{name:'zhang'})
delete n
// 如果节点有关系，需要先删除关系
```

# Set
```java
match (n:Person{name:'li'})
set n.name='lzw',n.age=28
return n.name,n.title,n.age

match (n:Person{name:'lzw'})
set n +={sex:1}
return n.name,n.title,n.age,n.sex
```

# Remove
```java
MATCH (n:Employee{employeeID:123})
remove n.firstName
```

# Foreach
```java
MATCH p =(begin)-[*]->(END )
WHERE begin.name = 'A' AND END .name = 'D'
FOREACH (n IN nodes(p)| SET n.marked = TRUE )
```

# UNION ALL
```java
MATCH (n:Actor)
RETURN n.name AS name
UNION ALL MATCH (n:Movie)
RETURN n.title AS name
```


# LOAD CSV
```java
LOAD CSV FROM 'https://neo4j.com/docs/cypher-manual/3.5/csv/artists.csv' AS line
CREATE (:Artist { name: line[1], year: toInteger(line[2])})

LOAD CSV WITH HEADERS FROM 'https://neo4j.com/docs/cypher-manual/3.5/csv/artists-with-headers.csv' AS line
CREATE (:Artist { name: line.Name, year: toInteger(line.Year)})
```


