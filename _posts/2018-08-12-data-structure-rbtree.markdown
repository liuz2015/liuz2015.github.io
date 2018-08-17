---
layout:     post
title:      "搞懂红黑树"
subtitle:   "领略数据结构之美"
date:       2018-08-12
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 数据结构
    - 二叉查找树
    - 红黑树
---

> 你不得不感叹红黑树是这么的有趣和美妙。

## 目录
- [简介](#简介)
- [性质](#性质)
- [结构](#结构)
- [旋转](#旋转)
- [插入](#插入)
- [删除](#删除)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

**红黑树**是一种“平衡”的二叉查找树。二叉查找树大家应该都知道，在这里不做赘述。

我们来看看“平衡”这两个字是什么意思。对于一棵普通的二叉查找树来说，假设它一共有n个节点，那么当我们进行查找操作时，当然希望最坏情况下的时间复杂度也仅为O(lgn)。不过，这样显然是需要一些前提条件的，那就是这棵树刚好长得比较“漂亮”，每个节点基本都有左右子节点，所有节点分布得很“平衡”，使得树的高度为lgn；有些树如果长成了“歪瓜裂枣”，比如一棵树的节点只有右子节点，树的高度就是n，那这棵树和链表又有什么区别呢？也就是说，树的操作的执行快慢，跟树“长”成什么样，即跟树的高度密切相关，高度低，执行快；高度高，执行慢。而在实际情况中，一棵普通的二叉树经过插入和删除等操作后，是无法控制自己长成什么样的，那么操作执行的快慢也就无法保证了。

于是乎人们便想出了一个方案，能不能设计一种能够自动维持自身平衡的二叉树，使得经过插入、删除等等操作后，这棵树能够维持自身的“平衡”，能够保证在最坏情况下的时间复杂度为O(lgn)。这类树就被称为**自平衡二叉查找树**，而红黑树就是其中的一种。

## 性质

满足以下的性质的二叉查找树，就是一棵红黑树。

**红黑树的五个性质：**

- 每个节点是红色，或是黑色。
- 根节点是黑色。
- 每个叶节点（NIL节点，空节点）是黑色。
- 如果一个节点是红色节点，则它的两个子节点都是黑色。
- 对每个节点，从该节点到其子孙节点的所有路径都包含相同数目的黑色节点。

看了以上性质，我们可能还是云里雾里，那我们就一条一条慢慢看。

第一条性质告诉我们，节点有颜色这个属性，而且颜色只能是红色或者黑色中的一种，所以叫红黑树嘛哈哈。颜色属性是普通的二叉树节点没有的，是其他性质的基础。

第二、三两条性质对根节点和叶子节点的颜色做了规定，这里注意一下这个叶子节点是指最下层节点的空子节点，

第四、五两条性质非常重要，是红黑树之所以能够维持“平衡”的基础。第四条告诉我们，红节点只能有黑子节点，也就是说从某节点出发到其子孙节点的任意一条路径不可能有连续两个红色节点；第五条告诉我们，从某节点出发到其子孙节点的任意一条路径上的黑节点数目相同。这两句话连起来，我们可以得到，一棵红黑树没有一条路径会大于其他路径的两倍，因此，红黑树是**接近平衡**的。

在红黑树的所有操作中，我们都要时刻保证其符合以上性质，从而维持一棵红黑树。

## 结构

树可以使用节点的方式来进行，那么红黑树的节点是什么样的呢？

首先，由于是二叉查找树，因此红黑树的节点同样也需要有值、父节点、左子节点、右子节点四个域；此外，红黑树的颜色性质，使得其要有一个颜色域来标记该节点为红色或黑色。

节点结构代码如下：

```cpp
struct Node
{
	int key;
	char color;
	Node* parent;
	Node* left;
	Node* right;
};
```

此外，为了代码上更好的实现红黑树，我们还定义了一些需要使用到的变量，并进行初始化，其中NIL节点是一个哨兵，指代叶子节点。

```cpp
const char red = 'R';
const char black = 'B';
Node* root = NULL;
Node* nil = NULL;

void init()//初始化哨兵和根节点
{
	nil = new Node;
	nil->color = black;
	root = nil;
}
```

## 旋转

现在我们需要思考如何去维持一棵红黑树的平衡了。如果一棵树“歪”了要怎么办？有同学说了，“歪”了那就把它再摆摆正呗。对，当然要摆正，怎么去摆正呢，这里就要介绍**旋转**这个操作了。

很多同学看到旋转这两个字就觉得有点头痛了，一棵好好的树，咋还要旋转呢？其实我们旋转的不是树，而是树里面的节点，把节点转一下，就把树给摆正了，怎么旋转节点呢？

先介绍**左旋**，如下图所示：

![左旋](https://gss1.bdstatic.com/-vo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike80%2C5%2C5%2C80%2C26/sign=890eaf031dd8bc3ed2050e98e3e2cd7b/86d6277f9e2f0708cdb73039ed24b899a901f25d.jpg)

从图中可以看到，pivot和Y两个节点的位置发生了变化，经过了一个逆时针的旋转（用你的左手去旋转节点，博主就这么记左旋）。经过左旋后，我们可以发现pivot从Y的父节点变成了Y的左子树，而这棵子树的高度也就产生了变化，通过这样的旋转操作，确实可以达到改变子树高度的目的。而且，最重要的一点是，旋转之后，二叉查找树的性质，也就是**值的大小关系与节点位置仍旧是正确**的。同时，b节点的红黑树性质5中的黑节点数目维持不变。

这时候你可能会开始想，那么我什么时候去左旋呢？是不是要左旋好多次？怎么知道左旋之后是否平衡了呢？

不急不急，左旋只是维持红黑树（其实不止红黑树，其他自平衡树也会用到）的一个基本操作而已，调整平衡的过程会在后面接着讲，我们先把左旋这个操作在代码上实现。看图可知，左旋这个操作不仅需要改变pivot和Y节点的相关值，也需要修改P和b节点的相关值，对其他的节点不会产生影响，不用理会。

左旋代码如下：

```cpp
Node* leftRotate(Node* node)
{
	Node* r = node->right;
	node->right = r->left;
	if (r->left!=nil)
	{
		r->left->parent = node;
	}
	r->left = node;
	r->parent = node->parent;
	node->parent = r;
	if (r->parent!=nil)
	{
		if (r->key>r->parent->key)
		{
			r->parent->right = r;
		}
		else
		{
			r->parent->left = r;
		}
	}
	else
	{
		root = r;
	}
	return r;
}
```

看了左旋后，右旋只不过就是左旋的镜像操作，在此不做赘述。

![右旋](https://gss3.bdstatic.com/7Po3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike80%2C5%2C5%2C80%2C26/sign=efcb430708f3d7ca18fb37249376d56c/cdbf6c81800a19d8cbd7803c37fa828ba61e4636.jpg)

右旋代码如下：

```cpp
Node* rightRotate(Node* node)
{
	Node* r = node->left;
	node->left = r->right;
	if (r->right!=nil)
	{
		r->right->parent = node;
	}
	r->right = node;
	r->parent = node->parent;
	node->parent = r;
	if (r->parent != nil)
	{
		if (r->key>r->parent->key)
		{
			r->parent->right = r;
		}
		else
		{
			r->parent->left = r;
		}
	}
	else
	{
		root = r;
	}
	return r;
}
```

## 插入

现在我们接着思考怎么去维持一棵树的平衡。对于树的诸多操作来说，只有插入和删除操作会修改树的节点，影响树的平衡；所以，我们只需要在进行这两个操作的时候，对树进行调整，使得在操作之后树是平衡的。

对于红黑树而言，我们要做的就是在插入和删除操作后，使得树仍能**满足红黑树的五个性质**即可。强调一下，从这里开始，我们进行调整的时候不要去想着树的高度之类的，我们只需要做到一切以红黑树的五个性质为准。

我们现在来看插入操作。

首先，我们需要实现基本的二叉查找树的插入功能，即根据新节点的值来找到插入的位置并插入。**为了保持性质5，我们将新插入的节点设为红色。为了保持性质3，将新节点的子节点设为nil节点。**

代码实现如下：

```cpp
void insert(Node*& root, int key)
{
	if (root==nil)
	{
		root = new Node;
		root->key = key;
		root->color = black;
		root->parent = root->left = root->right = nil;
		return;
	}
	Node* node = root;
	Node* p = node->parent;
	while (node!=nil)
	{
		p = node;
		if (key<node->key)
		{
			node = node->left;
		}
		else if (key>node->key)
		{
			node = node->right;
		}
		else
		{
			return;
		}
	}
	node = new Node;
	node->key = key;
	node->color = red;//将新插入的节点颜色设为红色
	node->parent = p;
	node->left = node->right = nil;
	if (key>p->key)
	{
		p->right = node;
	}
	else
	{
		p->left = node;
	}
	insertFix(node);//插入后进行调整
	return;
}
```

将节点插入后，我们来检查这个数是否满足五个性质，可以发现，有可能不满足性质4或性质2，违背性质2的情况就是插入的节点是根节点，这时候只需将其涂黑即可，这里不展开了，我们主要考虑新节点的颜色是红色，而它的父节点也是红色的情况，这时候我们就需要进行调整来维持性质4。针对节点和它的父节点都是红色的问题，这里需要分几种情况进行讨论。

**第一种情况**

该节点的叔叔节点也是红色，而父节点是爷爷节点的左子节点。这时候，我们将新节点的爸爸和叔叔节点都涂成黑色，性质4便满足了，但是——性质5被破坏了，我们需要把爷爷节点（一定是黑色的，因为爸爸是红色的）涂成红色，这时候，性质5是满足的，不过聪明的你们会想到，爷爷变成红色后是否还能满足性质4呢？是的，这时候，我们就讲原先要考虑的性质4问题从该节点转移到爷爷节点上。

**第二种情况**

该节点的叔叔节点是黑色，节点本身是父节点的左子节点，而父节点是爷爷节点的左子节点。我们先将父节点涂黑，将爷爷节点涂红，该节点的性质4就满足了，但是这时候性质5被破坏了，叔叔节点所在的路径少了一个黑色，怎么办？旋转！我们将父节点和爷爷节点进行右旋，父节点占了原来爷爷节点的位置，性质4没有被破坏的同时使得性质5被满足，再次确认，五条性质都被满足，收工！

**第三种情况**

该节点的叔叔节点是黑色，父节点是爷爷节点的左子节点，而该节点本身是父节点的右子节点。这时候和第二种情况的差别就是该节点是其父节点的右子节点了，这时候，如果还是按照第二种情况去调整，你可以画一下，最后依旧无法满足性质4。这时候怎么办呢？还是旋转，通过旋转把这个右子节点变成左子节点嘛，对该节点和其父节点进行左旋，旋转后，该节点移动到原先父节点的位置，而父节点则移动到左子节点的位置，这时候，就变成第二种情况了不是吗。

以上三种就是所有可能面临的情况了吗？当然不是，你发现上述情况父节点都是爷爷节点的左子节点，那如果是右子节点怎么办呢，和上述情况可以进行类比，就留给你去实现吧，这里就不赘述了。

![插入图解](/img/in-post/data-structure-rbtree/rbtree-insert.png)

代码实现如下：

```cpp
void insertFix(Node*& x)
{
	while (x->parent!=nil&&x->parent->color!=black)
	{
		if (x->parent==x->parent->parent->left)
		{
			if (x->parent->parent->right->color==red)//case1:叔叔是红色
			{
				x->parent->color = black;
				x->parent->parent->color = red;
				x->parent->parent->right->color = black;
				x = x->parent->parent;
			}
			else
			{
				if (x==x->parent->left)//case2:叔叔是黑色，x是左孩子
				{
					x->parent->color = black;
					x->parent->parent->color = red;
					x = rightRotate(x->parent->parent);
				}
				else//case3:叔叔是黑色，x是右孩子
				{
					x->color = black;
					x->parent->parent->color = red;
					x = leftRotate(x->parent);
					x = rightRotate(x->parent);
				}
			}
		}
		else
		{
			if (x->parent->parent->left->color == red)
			{
				x->parent->color = black;
				x->parent->parent->color = red;
				x->parent->parent->left->color = black;
				x = x->parent->parent;
			}
			else
			{
				if (x == x->parent->right)
				{
					x->parent->color = black;
					x->parent->parent->color = red;
					x = leftRotate(x->parent->parent);
				}
				else
				{
					x->color = black;
					x->parent->parent->color = red;
					x = rightRotate(x->parent);
					x = leftRotate(x->parent);
				}
			}
		}
	}
	if (x->parent==nil)
	{
		x->color = black;
		root = x;
	}
	return;
}

```

## 删除

我们再来看删除操作。首先，还是实现基本的二叉查找树的删除功能。这里简要说明一下删除功能的实现，在找到要删除的节点后，如果这个节点只有一个子节点或没有子节点，就删除它，并把子节点（或叶子节点）移动到被删节点的位置上；如果有左右子节点，则不会直接删除这个节点，而是找到该节点的**后继**节点（右子树中最小的那个节点），将后继节点的值赋给该节点，然后删除后继节点，并将后继节点的子节点（或叶子节点）移动到后继节点的位置上。

代码如下：

```cpp
void del(Node*& root, int key)
{
	Node* temp = root;
	while (temp!=nil&&temp->key!=key)
	{
		if (key>temp->key)
		{
			temp = temp->right;
		}
		else
		{
			temp = temp->left;
		}
	}
	if (temp==nil)
	{
		return;
	}
	Node* y = temp;
	if (temp->right!=nil&&temp->left!=nil)//寻找后继节点
	{
		y = y->right;
		while (y->left != nil)
		{
			y = y->left;
		}
	}
	Node* x = nil;
	if (y->left==nil&&y->right==nil)
	{
		if (y->parent == nil)
		{
			root = x;
			return;
		}
		else
		{
			x->parent = y->parent;
			if (y->key>y->parent->key)
			{
				y->parent->right = x;
			}
			else
			{
				y->parent->left = x;
			}
		}
	}
	else
	{
		if (y->left != nil)
		{
			x = y->left;
		}
		else
		{
			x = y->right;
		}
		if (y->parent == nil)
		{
			root = x; 
			root->parent = nil;
		}
		else
		{
			x->parent = y->parent;
			if (y->key>y->parent->key)
			{
				y->parent->right = x;
			}
			else
			{
				y->parent->left = x;
			}
		}
	}
	if (y!=temp)
	{
		temp->key = y->key;
	}
	if (y->color==black)
	{
		delFix(x);//删除后调整
	}
	y = NULL;
	return;
}
```
删除一个节点后，如果被删节点是红色节点，树的性质不会被破坏，不需要调整；如果被删节点是黑色节点，那么性质5就被破坏了，也就是说少了一个黑节点，需要调整使得重新满足性质5。

当节点被删除后（注意这个被删的节点是指被从树中移除的那个节点），它的子节点会被移动到原先被删节点的位置上，这时候，我们将移动到被删位置上的这个节点（后面称为补充节点）加上一层黑色，这时候补充节点的颜色为红黑色或者黑黑色，性质5可以被满足。然而，这时候性质1或性质2会被破坏。如果补充节点是红黑色，那么直接将其变为黑色，即可满足所有性质。如果为黑黑色，且不是根节点（如果是根节点则直接设为黑色），我们需要进一步调整，在保持性质5的前提下，将这层黑色进行转移，直到黑色转移到红黑色节点或根节点上。下面我们分情况展开讨论。

**第一种情况**

补充节点是父节点的左子节点，兄弟节点为红色。这时候，父节点必然是黑色的，我们将父节点和兄弟节点的颜色互换，并对其做一次左旋，性质4和性质5都没有被破坏使得父节点颜色变为红色，兄弟节点变成了爷爷节点，兄弟节点的左子节点变成了新的兄弟节点，颜色为黑色。但是这时候补充节点依旧为黑黑色，不满足性质1。然后我们需要进入其他情况进行考虑。

**第二种情况**

补充节点是父节点的左子节点，兄弟节点为黑色，兄弟节点的左右子节点都为黑色。将补充节点的多余黑色和兄弟节点的黑色向上转移给父节点，这时候性质4和性质5都能保持，我们只要关注父节点是否能满足性质1，若不能，则开始新一轮循环，解决父节点的性质1问题。

**第三种情况**

补充节点是父节点的左子节点，兄弟节点为黑色，兄弟节点的左子节点为黑色，右子节点为红色。先将兄弟节点和兄弟节点的右子节点调换颜色，这时候，性质5被破坏，兄弟节点的左子节点少了个黑节点，性质4可能被破坏，父节点可能是红色的，而兄弟节点此时是红色的。于是将补充节点的多余黑色转移给兄弟节点，即把兄弟节点染成黑色，性质4满足了，且兄弟节点的左子节点黑节点数目恢复，但是性质5还是没有满足，兄弟节点的右子节点多了一个黑节点，补充节点路径上也少了一个黑节点，将兄弟节点和父节点左旋，性质5即可被满足，调整完成。

**第四种情况**

补充节点是父节点的左子节点，兄弟节点为黑色，兄弟节点的右子节点为黑色，左子节点为红色。该情况域第三种情况区别就在兄弟节点的子节点颜色上，将兄弟节点和左子节点互换颜色，然后对其进行右旋，即可得到第三种情况，且性质4、5没有被破坏。

以上为补充节点是父节点的左子节点时的所有情况，当为右子节点时，可以类比实现，不做赘述。

![删除图解](/img\in-post\data-structure-rbtree\rbtree-delete.png)

代码如下：

```cpp
void delFix(Node*& x)//没有哨兵的话，删除恢复时x为NULL就不好处理
{
	while (x->parent!=nil&&x->color==black)
	{
		if (x==x->parent->left)
		{
			Node* w = x->parent->right;
			if (w->color==red)//case1:兄弟w为红色，父亲为黑色，w孩子为黑色
			{
				w->color = black;
				x->parent->color = red;
				leftRotate(x->parent);
				w = x->parent->right;
			}
			else
			{
				if ((w->left == nil || w->left->color == black) && (w->right == nil || w->right->color == black))
					//case2:w为黑色，且w孩子都为黑色
				{
					w->color = red;
					x = x->parent;
				}
				else
				{
					if (w->right == nil || w->right->color == black)//case3:w为黑色，w左孩子为红色，右孩子为黑色
					{
						w->color = red;
						w->left->color = black;
						rightRotate(w);
						w = x->parent->right;
					}
					//case4:w为黑色，w右孩子为红色
					w->right->color = black;
					w->color = w->parent->color;
					w->parent->color = black;
					leftRotate(w->parent);
					x = root;
				}
			}
		}
		else
		{
			Node* w = x->parent->left;
			if (w->color == red)
			{
				w->color = black;
				x->parent->color = red;
				rightRotate(x->parent);
				w = x->parent->left;
			}
			else
			{
				if ((w->left == nil || w->left->color == black) && (w->right == nil || w->right->color == black))
				{
					w->color = red;
					x = x->parent;
				}
				else
				{
					if (w->left == nil || w->left->color == black)
					{
						w->color = red;
						w->right->color = black;
						leftRotate(w);
						w = x->parent->left;
					}
					w->left->color = black;
					w->color = w->parent->color;
					w->parent->color = black;
					rightRotate(w->parent);
					x = root;
				}
			}
		}
	}
	x->color = black;
}
```

## 总结

从使查找树平衡的思想出发，出现了许多的平衡树，红黑树是其中的一种，其他的还有：AVL树、SBT、伸展树、TREAP等。

其中AVL树是**高度平衡**的树（红黑树是接近平衡），对每个节点，其左子树和右子树的高度差至多为1。在AVL树的节点中增加了高度属性，并利用高度来维持树的平衡。

但是红黑树的应用更加广，Java和C++库函数有很多地方都使用红黑树作为平衡树的实现。有同学想问为什么使用红黑树，而不是AVL树呢。

**考虑如下：**

1. 如果操作引起了树的不平衡，在维护树的过程中，avl树需要进行高度计算，红黑树则需要进行红黑性质维护，这里的消耗差不多。但是对于旋转来说，在插入时，AVL和RB-Tree都是最多只需要2次旋转操作，即两者都是O(1)；但是在删除node时，最坏情况下，AVL需要维护从被删node到root这条路径上所有node的平衡性，因此需要旋转的量级O(logN)，而RB-Tree最多只需3次旋转，只需要O(1)的复杂度。

2. AVL的结构相较RB-Tree来说更为平衡，在插入和删除node更容易引起Tree的unbalance，因此在大量数据需要插入或者删除时，AVL需要rebalance的频率会更高。因此，RB-Tree在需要大量插入和删除node的场景下，效率更高。不过，由于AVL高度平衡，因此AVL的search效率更高。

大家还可以去看看Java或C++源码中红黑树的实现及应用场景，可以有一些更深入的体会哦~

## 参考资料
- 《算法导论》
- [红黑树_百度百科](https://baike.baidu.com/item/%E7%BA%A2%E9%BB%91%E6%A0%91/2413209?fr=aladdin)
- [史上最清晰的红黑树讲解](https://github.com/CarpenterLee/JCFInternals/blob/master/markdown/5-TreeSet%20and%20TreeMap.md)
- [为什么STL和linux都使用红黑树作为平衡树的实现？](https://www.zhihu.com/question/20545708)

