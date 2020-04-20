上一篇文章中，展示了怎么以Height作为平衡判断条件来实现AVL树。这一篇文章中，我们将以平衡因子bf作为平衡判断条件，来实现AVL树。在正文开始前，有几点想要吐槽的地方：

- 《大话数据结构》和严蔚敏的数据结构书中展示的代码都是以bf作为判断条件的，但是他们只给出了insert代码，看不见remove的代码（至少书上没有，不知道有没有什么附加链接可以下载到有remove代码的源码。。。）
- 以bf作为平衡判断条件，是在是太难写了，一不小心就万劫不复。
- 承接上一点，如果掌握了以bf作为平衡判断条件的代码的写法，才能算真正理清楚了AVL树的旋转。

下面开始正文吧。

## 基本代码

先定义node，以及平衡因子的三个状态。

```c++
#define LH 1
#define EH 0
#define RH -1

struct node {
	int data;
	int bf;
	node * left, *right;
	node(int data) :data(data), bf(EH), left(nullptr), right(nullptr) {}
};
```

平衡因子bf，实际上就是左右子树的高度差。由于AVL树的左右子树高度差不超过2，所以有三种情况：

- LH，左子树高于右子树
- EH，左右子树高度相同
- RH，右子树高于左子树

## 旋转

旋转的代码主体上不变。只不过，不同于用height作为平衡判断条件的那个版本，这个版本我们不在选择的代码里面更新bf，因为更新bf的工作非常麻烦，需要分情况进行讨论，在旋转的代码里无法很好的实现更新。

```c++
static void RotateR(node ** pRoot)
{
	node * root = *pRoot;
	node * pivot = (*pRoot)->left;

	root->left = pivot->right;
	pivot->right = root;
	*pRoot = pivot;
}

static void RotateL(node ** pRoot)
{
	node * root = *pRoot;
	node * pivot = (*pRoot)->right;

	root->right = pivot->left;
	pivot->left = root;
	*pRoot = pivot;
}

static void RotateLR(node ** pRoot)
{
	RotateL(&((*pRoot)->left));
	RotateR(pRoot);
}

static void RotateRL(node ** pRoot)
{
	RotateR(&((*pRoot)->right));
	RotateL(pRoot);
}
```

## Insert函数

首先是同样的主体框架：

```c++
void Insert(node ** pPtr, int data)
{
	node * n = *pPtr;
	if(!n)
		*pPtr = new node(data);
	else if(data < n->data)
	{
		Insert(&n->left, data);
        //todo: 检查失衡并旋转
	}
	else if(n->data < data)
	{
		Insert(&n->right, data);
		//todo: 检查失衡并旋转
	}
	UpdateHeight(*pPtr);
}
```



## 源代码

```c++
#define LH 1
#define EH 0
#define RH -1

struct node {
	int data;
	int bf;
	node * left, *right;
	node(int data) :data(data), bf(EH), left(nullptr), right(nullptr) {}
};

static void RotateR(node ** pRoot)
{
	node * root = *pRoot;
	node * pivot = (*pRoot)->left;

	root->left = pivot->right;
	pivot->right = root;
	*pRoot = pivot;
}

static void RotateL(node ** pRoot)
{
	node * root = *pRoot;
	node * pivot = (*pRoot)->right;

	root->right = pivot->left;
	pivot->left = root;
	*pRoot = pivot;
}

static void RotateLR(node ** pRoot)
{
	RotateL(&((*pRoot)->left));
	RotateR(pRoot);
}

static void RotateRL(node ** pRoot)
{
	RotateR(&((*pRoot)->right));
	RotateL(pRoot);
}

static node * FindMin(node * root)
{
	node * n = root;
	while (n->left)
		n = n->left;
	return n;
}

static void InsertBalanceLeft(node ** pPtr)
{
	node * n = *pPtr;
	node * l = n->left;
	switch (l->bf)
	{
	case LH:
		n->bf = l->bf = EH;
		RotateR(pPtr);
		break;
	case RH:
		node * lr = l->right;
		switch (lr->bf)
		{
		case LH:
			n->bf = RH;
			l->bf = EH;
			break;
		case EH:
			n->bf = l->bf = EH;
			break;
		case RH:
			n->bf = EH;
			l->bf = LH;
			break;
		}
		lr->bf = EH;
		RotateLR(pPtr);
		break;
	}
}

static void InsertBalanceRight(node ** pPtr)
{
	node * n = *pPtr;
	node * r = n->right;
	switch (r->bf)
	{
	case RH:
		n->bf = r->bf = EH;
		RotateL(pPtr);
		break;
	case LH:
		node * rl = r->left;
		switch (rl->bf)
		{
		case RH:
			n->bf = LH;
			r->bf = EH;
			break;
		case EH:
			n->bf = r->bf = EH;
			break;
		case LH:
			n->bf = EH;
			r->bf = RH;
			break;
		}
		rl->bf = EH;
		RotateRL(pPtr);
		break;
	}
}

void Insert(node ** pPtr, int data, bool * taller)
{
	node * n = *pPtr;
	if (!n)
	{
		*pPtr = new node(data);
		*taller = true;
	}
	else if (data < n->data)
	{
		Insert(&n->left, data, taller);
		if (*taller)
		{
			switch (n->bf)
			{
			case LH:
				InsertBalanceLeft(pPtr);
				*taller = false;
				break;
			case EH:
				n->bf = LH;
				*taller = true;
				break;
			case RH:
				n->bf = EH;
				*taller = false;
				break;
			}
		}
	}
	else if (n->data < data)
	{
		Insert(&n->right, data, taller);
		if (*taller)
		{
			switch (n->bf)
			{
			case LH:
				n->bf = EH;
				*taller = false;
				break;
			case EH:
				n->bf = RH;
				*taller = true;
				break;
			case RH:
				InsertBalanceRight(pPtr);
				*taller = false;
				break;
			}
		}
	}
}

static void RemoveBalanceLeft(node ** pPtr, bool * lower)
{
	node * n = *pPtr;
	node * l = n->left;
	*lower = false;
	switch (l->bf)
	{
	case EH:
		n->bf = LH;
		l->bf = RH;
		RotateR(pPtr);
		break;
	case LH:
		n->bf = l->bf = EH;
		RotateR(pPtr);
		*lower = true;
		break;
	case RH:
		node * lr = l->right;
		switch (lr->bf)
		{
		case LH:
			n->bf = RH;
			l->bf = EH;
			break;
		case EH:
			n->bf = l->bf = EH;
			break;
		case RH:
			n->bf = EH;
			l->bf = LH;
			break;
		}
		lr->bf = EH;
		RotateLR(pPtr);
		*lower = true;
		break;
	}
}

static void RemoveBalanceRight(node ** pPtr, bool * lower)
{
	node * n = *pPtr;
	node * r = n->right;
	*lower = false;
	switch (r->bf)
	{
	case EH:
		n->bf = RH;
		r->bf = LH;
		RotateL(pPtr);
		break;
	case RH:
		n->bf = r->bf = EH;
		RotateL(pPtr);
		*lower = true;
		break;
	case LH:
		node * rl = r->left;
		switch (rl->bf)
		{
		case RH:
			n->bf = LH;
			r->bf = EH;
			break;
		case EH:
			n->bf = r->bf = EH;
			break;
		case LH:
			n->bf = EH;
			r->bf = RH;
			break;
		}
		rl->bf = EH;
		RotateRL(pPtr);
		*lower = true;
		break;
	}
}

void Remove(node ** pPtr, int data, bool * lower)
{
	node * n = *pPtr;
	if (!n)
	{
		*lower = false;
		return;
	}
	if (data < n->data)
	{
		Remove(&n->left, data, lower);
		if (*lower)
		{
			switch (n->bf)
			{
			case LH:
				n->bf = EH;
				*lower = true;
				break;
			case EH:
				n->bf = RH;
				*lower = false;
				break;
			case RH:
				RemoveBalanceRight(pPtr, lower);
				break;
			}
		}
	}
	else if (n->data < data)
	{
		Remove(&n->right, data, lower);
		if (*lower)
		{
			switch (n->bf)
			{
			case LH:
				RemoveBalanceLeft(pPtr, lower);
				break;
			case EH:
				n->bf = LH;
				*lower = false;
				break;
			case RH:
				n->bf = EH;
				*lower = true;
				break;
			}
		}
	}
	else if (n->left && n->right)
	{
		node * victim = FindMin(n->right);
		n->data = victim->data;
		Remove(&n->right, n->data, lower);
		if (*lower)
		{
			switch (n->bf)
			{
			case LH:
				RemoveBalanceLeft(pPtr, lower);
				break;
			case EH:
				n->bf = LH;
				*lower = false;
				break;
			case RH:
				n->bf = EH;
				*lower = true;
				break;
			}
		}
	}
	else
	{
		node * old = n;
		*pPtr = n->left ? n->left : n->right;
		delete old;
		*lower = true;
	}
}
```

