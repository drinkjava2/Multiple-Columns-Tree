# Multiple-Columns-Tree & Sorted-Unlimitation-Depth-Tree
A new solution for save hierarchical data (Tree structure) in Database  
(For Chinese version see: README-中文版)
  
Currently there are 4 common tree structure database storage patten, but they all have some problems:
1)Adjacency List: Only record parent node. Advatage is simple, shortage is hard to access child nodes tree, need send lots SQL to database.
2)Path Enumerations：Using a string to record path info. Advantage is easy to query, shortage is hard do insert operation, need modify lots path strings when insert a new record.
3)Closure Table：Using another table to record path info, Advatage is easy to query and maintain, shortage is take too much space.
4)Nested Sets：Record Left and Right Nodes, Advatange is easy to query/delete/insert, shortage is too complex.
All of above patten has a same problem: not clearly show the whole tree structure in database.
Here I invented 2 new methods to store hierarchical data (Tree structure) in Database (Note: I'm not sure if someone else already invented these method before, maybe I did not spend enough time to search on internet).

## Method1 
I call it "simple multiple colums tree" patten, it's similar lie Path Enumerations patten, but not exactly same, the differenc is it use lots database columns to store a position mark (1 or null), see below picture((https://github.com/drinkjava2/Multiple-Columns-Tree/blob/master/treemapping.jpg):
![image](treemapping.jpg)

line is a sorted number record the line code, column name is the depth of the node, "C1" means it's a root node.

To access tree from database,  using below SQL:
```
1.Query or delete all child nodes tree, for node line=X, level=cY
select *(or delete) from tb where 
  line>=X and line<(select min(line) from tb where line>X and  (cY=1 or c(Y-1)=1 or c(Y-2)=1 ... or c1=1))
For example, get D child nodes tree:
select * from tb where line>=7 and line< (select min(line) from tb where line>7 and (c2=1 or c1=1)) 
Delete D child nodes tree:
delete from tb where line>=7 and line< (select min(line) from tb where line>7 and (c2=1 or c1=1)) 

Only get direct child nodes of D:
select * from tb where line>=7 and c3=1 and line< (select min(line) from tb where line>7 and (c2=1 or c1=1)) 

2.Query root node for node line=X, level=cY:
select * from tb where line=(select max(line) from tb where line<=X and c1=1)
For example, query root node for node I:
select * from tb where line=(select max(line) from tb where line<=12 and c1=1) 

3.Query parent node for node line=X, level=cY:
select * from tb where line=(select max(line) from tb where line<X and c(Y-1)=1)
For example get parent node for node L:
select * from tb where line=(select max(line) from tb where line<11 and c3=1) 

4.Query all parent nodes for node line=X, level=cY:
select * from tb where line=(select max(line) from tb where line<X and c(Y-1)=1)
union select * from tb where line=(select max(line) from tb where line<X and c(Y-2)=1)
...
union select * from tb where line=(select max(line) from tb where line<X and c1=1)
For example, get all parent nodes for node I:
select * from tb where line=(select max(line) from tb where line<12 and c2=1)
union  select * from tb where line=(select max(line) from tb where line<12 and c1=1) 
 
5.Insert a new node, for example, insert a new node between J and K:
update tb set line=line+1 where line>=10;
insert into tb (line,id,c4) values (10,'T',1)
Note: to avoid "update tb set line=line+1" lock all table, suggest add an "GroupID" column, all nodes within same root node share one groupID, for example:
update tb set line=line+1 where groupid=2 and line>=8;
insert into tb (groupid,line,c4) values (2, 8,'T')
by this way can improve performance.

```
Summary of method#1
Advatange of "simple multiple colums tree" patten:
1）Easy understand, the only patten can directly see the tree in database.  
2）Can very few SQL do Query, Insert, delete operation
3）Only need 1 table
4) Fit all type database

Shortage:
1) Has depth limitation because usually database allowed maximum column <1000, and for performance consideration, suggestion use this method within depth<100.
2) Sql is long, often need use n! "c9=1 or c8=1  or c7=1 ... or c1=1" type sql.
3) Hard to move nodes tree. Suitable for applications only often do increase/delete, very few moving nodes operations.
4) Take too much database space


## Method2
To improve it, use only one column instead of using many columns to record the depth level, by this improvement, it has no limitation of depth level and much simpler than method1, I give it a name "Sorted-Unlimitation-Depth-Tree", see below picture
(https://github.com/drinkjava2/Multiple-Columns-Tree/blob/master/treemappingv2.png) , please note all node are categorized in group, each group has a "END" tag with level value "0":
![image](treemappingv2.png)
```
To access tree from database, using below SQL
1.Query or delete all child nodes tree, for node line=X, level=Y, groupid=Z:
select * from tb2 where groupID=Z and 
  line>=X and line<(select min(line) from tb where line>X and level<=Y and groupID=Z)
For example, get D child nodes tree:
select * from tb2 where groupID=1 and 
  line>=7 and line< (select min(line) from tb2 where groupid=1 and line>7 and level<=2)
Delete D child nodes tree:
delete from tb2 where groupID=1 and 
  line>=7 and line< (select min(line) from tb2 where groupid=1 and line>7 and level<=2)

Only get direct child nodes of D:
select * from tb2 where groupID=1 and 
  line>=7 and level=3 and line< (select min(line) from tb2 where groupid=1 and line>7 and level<=2) 

2.Query root node for any node with groupid=Z
select * from tb2 where groupID=Z and level=1 

3.Query parent node for node line=X, level=Y, groupid=Z:
select * from tb2 where groupID=Z and 
  line=(select max(line) from tb2 where groupID=Z and line<X and level=(Y-1))
For example get parent node for node L:
select * from tb2 where groupID=1 
  and line=(select max(line) from tb2 where groupID=1 and line<11 and level=3) 

4.Query all parent nodes for node line=X, level=Y, groupid=Z:
select * from tb2 where groupID=Z and 
  line=(select max(line) from tb2 where groupID=Z and line<X and level=(Y-1))
union select * from tb2 where groupID=Z and 
  line=(select max(line) from tb2 where groupID=Z and line<X and level=(Y-2))
...
union select * from tb2 where groupID=Z and 
  line=(select max(line) from tb2 where groupID=Z and line<X and level=1)
For example get parent node for node I:
select * from tb2 where groupID=1 and 
  line=(select max(line) from tb2 where groupID=1 and line<12 and level=2)
union  select * from tb2 where groupID=1 and 
  line=(select max(line) from tb2 where groupID=1 and line<12 and level=1)

5.Insert a new node, for example, insert a new node between J and K:
update tb2 set line=line+1 where  groupID=1 and line>=10;
insert into tb (groupid,line,id,level) values (1,10,'T',4);
```

Summary of method#2
Advatange：  
1） Has no depth level limitation
2） Simple, easy understand   
3） Easier than method#1 to do SQL do query/delete/insert operation
4） Only need 1 table
5） Fit all type database
6） Take very few database space

缺点有:  
Shortage:
1) Hard to move nodes tree. Suitable for applications only often do increase/delete, very few moving nodes operations.
