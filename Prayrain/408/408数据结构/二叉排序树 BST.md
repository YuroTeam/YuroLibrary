

---
#### 性质
![[Pasted image 20260820174035.png]]

自身及其子树结点值大小一定为：左<根<右，因此中序遍历可以得到一个结点值递增的有序序列

#### 结构体

![[Pasted image 20260820174324.png]]

BST的查找规则如下：
![[Pasted image 20260821022457.png]]
![[Pasted image 20260821023116.png]]

BST的ASL如下：
![[Pasted image 20260821023253.png]]

含有相同结点数的二叉树，树型越均衡的ASL越小
![[Pasted image 20260821023414.png]]![[Pasted image 20260821023440.png]]


#### 插入操作
![[Pasted image 20260821023519.png]]

插入操作基本和查找的原理一样，只是在查找的基础上加入了一些插入语句，时间复杂度相同


#### 删除操作
![[Pasted image 20260821023635.png]]
![[Pasted image 20260821023745.png]]

![[Pasted image 20260821023909.png]]


#### 创建操作
![[Pasted image 20260821024031.png]]

创建操作几乎就是循环插入

![[Pasted image 20260821024140.png]]