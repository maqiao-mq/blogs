这篇文章不适合于从未听说过AVL树的人，适合对AVL树有概念，但是在实现上有困难的人。

AVL树网上的博客一搜一大把，大部分都在讲AVL怎么左旋、右旋的，在这之后就没了，怎么将一个新的元素插入到树中，怎么判断AVL树失衡，怎么判断是哪种失衡，都没有好好提到，而且许多博客根本不讲怎么删除节点。所以，我查阅了各种资料，详细记录了关于AVL树在实现的各种细节，并给出完整的源代码。

如果读者是一个想要学习怎么写AVL树的人，建议按顺序阅读这篇文章，并且一边读一边跟着写代码。在阅读的同时，如果有不理解的地方，可以参考[这个网站](https://www.cs.usfca.edu/~galles/visualization/AVLtree.html)。
<!-- more -->

## 旋转
首先，最重要的当然是AVL树的旋转了。下面这幅图来自维基百科，把四种情况都详细展示了出来。至于旋转的原理，有兴趣的可以参考[wiki]([https://zh.wikipedia.org/wiki/%E6%A0%91%E6%97%8B%E8%BD%AC](https://zh.wikipedia.org/wiki/树旋转))，这里不进行解释。总之，在实现AVL树之前，需要先记住这四种旋转方式。

![](Tree_Rebalancing.png)

| 失衡情况 | 旋转方式 |
| -------- | -------- |
| LL       | R        |
| LR       | LR       |
| RR       | L        |
| RL       | RL       |

在网上能找到的资料中，AVL树的实现通常有两种版本，一种是使用height，也就是树高作为平衡判断条件；另一种是使用平衡因子balance factor（简写成bf，其实就是左右子树高差值）作为平衡判断条件。其中，使用height的版本要简单一些，我们将从它开始。

## 以height作为平衡判断条件
以height作为平衡条件的代码实现参考了github上的[这份代码](https://github.com/i-square/Data-Structure/blob/master/Chapter04/AVLTree.h)。

### 基本代码
实现AVL树，第一步自然是定义树的节点。节点的定义如下。出于简化的目的，node的key是一个int类型。node的height记录树高，left和right指针分别指向当前节点的左儿子和右儿子。

```c++
struct node{
    int data;	//key
    int height;	//树高
    node * left, * right;	//左右儿子
    node(int data):data(data), height(0), left(nullptr), right(nullptr){}
};
```

然后，我们需要两个辅助函数，分别用于计算和更新树高。

```c++
static int Height(node * root)
{
    return root?root->height:0;
}

static void UpdateHeight(node * root)
{
    root->height = max(Height(root->left), Height(root->right))+1;
}
```
### 实现树的旋转

现在，我们需要实现四种旋转。对于LL失衡，要做的是右旋。其代码如下。

![](LL.png)
```c++
static void RotateR(node ** pRoot)
{
    node * root = *pRoot;
    node * pivot = (*pRoot)->left;

    root->left = pivot->right;
    pivot->right = root;
    *pRoot = pivot;
    UpdateHeight(root);
    UpdateHeight(pivot);
}
```

树的一次旋转，会改变这棵树的根节点。一般情况下，我们当前旋转的这棵树是某棵树的子树。这样一来，对这棵子树进行了旋转以后，由于它的根节点改变了，所以需要更新它的父节点parent的left/right指针。所以，我们的RotateR函数接收的参数是node的二维指针，以方便对parent->left/right进行更新。也就是说，调用该函数需要将这棵子树的父节点的left/right指针的地址传递进来，即`&parent->left/right`。

RotateR函数的实现很简单。首先，参照右旋的示意图，对root和pivot的指针进行更新，然后，由于根节点变了，所以parent的指针也要进行更新。最后，旋转可能导致root和pivot的树高发生变化，所以需要对它们的树高进行更新。在更新树高时，注意先更新root、再更新pivot，这样才能保证树高更新的正确性。

依葫芦画瓢，RotateL的代码也很容易被实现出来。如下所示。

```c++
static void RotateL(node ** pRoot)
{
    node * root = *pRoot;
    node * pivot = (*pRoot)->right;

    root->right = pivot->left;
    pivot->left =  root;
    *pRoot = pivot;
    UpdateHeight(root);
    UpdateHeight(pivot);
}
```

RotateLR和RotateRL是RotateL和RotateR的不同组合，所以实现这两个函数，只需要按照不同去调用RotateL和RotateR即可，代码见下方。不过，按照下方代码这样实现，一次RotateLR/RotateRL会有两次函数调用，而且其中有些节点的指针会被重复更新，这样效率会稍微有些低下。对性能有更高要求的可以参照LR/RL的旋转方式，直接对各个节点的指针进行更新，而不是调用两次函数。

```c++
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

### Insert实现
Insert的实现有几个要点：

1. 虽然我们用的是int作为key，int类型支持`>`、`<`、`==`等比较方式。但是，为了代码的可扩展性更强，我们参考stl的map，只使用`<`作为大小的比较方式。换句话说，我们的代码假设key是严格偏序的。
2. 在insert并balance当前子树后，可能会导致上层失衡，如此一来，上层也必须进行balance。换句话说，balance是递归向上进行的，直到某一层没有出现失衡为止。
3. 在balance后，当前子树的树高可能发生变化，从而导致其到根节点路径上的所有节点的树高都可能发生变化，因此在insert后需要对这一路径上的所有节点的树高都进行更新。换句话说，更新树高也是递归向上的。

现在，我们来实现Insert。首先要解决的问题是怎么找insert的位置，由于AVL树也是一种BST树，所以它的左子树的值都比当前节点的值小，右儿子都比当前节点的值大。依据这一性质，就能写出Insert函数的大致框架：

```c++
//pPtr的含义与Rotate函数的参数相同，这里不再解释
//data是将要insert进来的key
void Insert(node ** pPtr, int data)
{
    //n即是当前子树的root节点
    node * n = *pPtr;
    //如果当前root节点为空，说明找到了要insert的地方了
    if(!n)
        *pPtr = new node(data);
    //只使用 < 符号进行比较
    else if(data < n->data)
    {
        Insert(&n->left, data);
        //todo: 检查失衡并旋转
    }
    //不支持重复key插入，这里就需要判断一下n->data<data。
    //如果支持重复key插入，这里直接一个else就行（没实证过，但理论上如此）。
    else if(n->data < data)
    {
        Insert(&n->right, data);
        //todo: 检查失衡并旋转
    }
    //balance会导致树高变化，所以路径上的每一个节点都需要更新树高
    UpdateHeight(*pPtr);
}
```

现在，剩下的问题就是判断是否出现失衡，出现了哪种类型的失衡，以及怎么对子树进行balance。

- 第一个问题，怎么判断失衡。很简单，根据定义，左右节点高度差大于等于2就失衡了。  
- 第二个问题，判断失衡类型。  
  - 对于`data<n->data`的情况，新的节点加入到了左子树，那么只可能出现LL和LR的失衡。那么怎么判断是LL还是LR呢？  
    - 第一种办法是比较新值data和n->left->data的大小。  
      - 如果`data<n->left->data`，那么新的节点在n->left的左子树中，所以是LL失衡。  
      - 反之就是LR失衡了。  
    - 第二种办法是比较n->left的左右子树的树高。  
      - 左子树树高大于等于右子树，就是LL失衡。  
      - 反之就是LR失衡。  
  - 对于`data>n->data`的情况，判断方式基本相同，这里不再赘述。  
- 第三个问题，怎么进行balance。这一点在开头已经提过了那四种情况，这里不再赘述。  

思考得比较深的读者，可能会发现上面的一个小问题。判断失衡类型的第二种方法中，左子树和右子树树高相同是怎么回事？事实上，在insert的过程中这种情况根本不会发生，这里列举出第二种方法，是为了给remove方法埋个伏笔。

根据上面的描述，我们可以补全Insert的代码了。

```c++
void Insert(node ** pPtr, int data)
{
    node * n = *pPtr;
    if(!n)
        *pPtr = new node(data);
    else if(data < n->data)
    {
        Insert(&n->left, data);
        //判断是否出现失衡
        if(Height(n->left)-Height(n->right)>=2)
        {
            //判断出现了哪种失衡，也可以注释中的代码进行判断
            //if(Height(n->left->left)>=Height(n->left->right))
            if(data < n->left->data)
                RotateR(pPtr);	//LL
            else
                RotateLR(pPtr);	//LR
        }
    }
    else if(n->data < data)
    {
        Insert(&n->right, data);
        if(Height(n->right)-Height(n->left)>=2)
        {
            //与左子树的判断类似，也可以注释中的代码进行判断
            //if(Height(n->right->right)>=Height(n->right->left))
            if(n->right->data < data)
                RotateL(pPtr);
            else
                RotateRL(pPtr);
        }
    }
    UpdateHeight(*pPtr);
}
```
###  Remove实现
remove的实现其实比insert要麻烦，难点在于在remove一个节点后，为了保持树左子树节点的值都比当前节点小，右子树节点的值都比当前节点大这一性质，还需要对树进行平衡操作。

所以，remove的代码实现主要分为两个部分：

1. 寻找对应的节点并进行删除操作。  
2. 删除后对树进行再平衡。  

先来说第一个问题，寻找合适的节点并进行删除操作。AVL树的删除一个节点的操作和BST的一样，具体如下：

1. 如果被删除节点是叶子节点，那么直接删除。  
2. 如果被删除节点只有左儿子或者右儿子节点，那么直接删除该节点，并把它的父亲与它的左/右儿子连在一起。  
3. 如果被删除节点左右儿子都有，那么选择它右儿子中最小的那个节点作为替死鬼，把这个替死鬼的值赋给当前节点，并把替死鬼节点删掉。当然，选择左儿子中最大的那个节点也是可行的。  

首先我们根据BST查找一个节点的方式，写出下面的框架代码：

```c++
void Remove(node ** pRoot, int data)
{
    node * n = *pRoot;
    if(!n)
        return;
    if(data < n->data)
    {
        Remove(&n->left, data);
        UpdateHeight(n);
        //todo: 检查失衡并旋转
    }
    else if(n->data < data)
    {
        Remove(&n->right, data);
        UpdateHeight(n);
        //todo: 检查失衡并旋转
    }
    else
    {
        //已经找到了满足条件的key对应的节点
        //todo: 删除节点并检查失衡和旋转
    }
}
```

接下来，补全删除节点的代码。首先，我们需要一个辅助函数来找到右儿子中最小的那个节点，代码如下。这个函数不断地沿着AVL树的左儿子进行移动，直到找到AVL树左下角的那个节点，即AVL树中最小的节点。

```c++
static node* FindMin(node * root)
{
    node * n = root;
    while(n->left)
        n = n->left;
    return n;
}
```

接下来，我们继续补全Remove的代码，代码如下。当找到对应的节点后，

- 如果这个节点是中间节点，则需要找到一个替死鬼节点，也就是它右子树中最小的那个节点，这可以通过FindMin函数来进行。在找到这个节点后，将它的值赋给当前节点，然后删除这个替死鬼节点。  
- 否则，当前节点是只有左/右儿子的节点或者叶子节点，我们只需要下面的短短三行代码就能将其删除。  

```c++
void Remove(node ** pRoot, int data)
{
    ...
    ...
    if(data < n->data){ Remove(&n->left, data); ... }
    else if(n->data < data)	{ Remove(&n->right, data); ... }
    else 
    {
        if(n->left && n->right)
        {
            //当前节点是中间节点
            //寻找替死鬼
            node * victim = FindMin(n->right);
            n->data = victim->data;
            //删除替死鬼节点
            Remove(&n->right, n->data);
            UpdateHeight(n);
            //todo: 检查失衡并旋转
        }
        else
        {
            //*pRoot指向的节点只有左/右儿子的节点，或者该节点是叶子节点
            node * old = n;
            *pRoot = n->left?n->left:n->right;
            delete old;
        }
    }
}
```

到现在为止，第一个大问题——寻找对应的节点并进行删除操作，已经被解决了。接下来是再平衡的问题。与Insert类似，这个问题细分为三个小问题：如何判断是否出现失衡，出现了哪种类型的失衡，以及怎么对子树进行balance。判断失衡当然是通过比较左右子树的高度差，大于等于2就失衡。对子树如何进行balance也已经在旋转那一小节讲过了。主要的问题是判断失衡类型，这也与Insert相似。

- 如果是在左子树中删除的节点，那么只可能出现RR和RL失衡。区分RR和RL失衡的办法是比较n->right的左右子树的树高：左子树树高大于等于右子树，就是RR失衡。反之就是RL失衡。
- 同样的道理，就能区分出LL和LR失衡了。

于是，我们可以写出BalanceRight和BalanceLeft的代码：

```c++
static void BalanceRight(node ** pPtr)
{
    node * n = *pPtr;
    if(Height(n->right)-Height(n->left)>=2)
    {
        if(Height(n->right->right)>=Height(n->right->left))
            RotateL(pPtr);
        else
            RotateRL(pPtr);
    }
}
static void BalanceLeft(node ** pPtr)
{
    node * n = *pPtr;
    if(Height(n->left)-Height(n->right)>=2)
    {
        if(Height(n->left->left)>=Height(n->left->right))
            RotateR(pPtr);
        else
            RotateLR(pPtr);
    }
}
```

接下来，就可以对Remove的代码进行最终补全了。

```c++
void Remove(node ** pPtr, int data)
{
    node * n = *pPtr;
    if(!n)
        return;
    if(data < n->data)
    {
        Remove(&n->left, data);
        UpdateHeight(n);
        BalanceRight(pPtr);
    }
    else if(n->data < data)
    {
        Remove(&n->right, data);
        UpdateHeight(n);
        BalanceLeft(pPtr);
    }
    else if(n->left && n->right)
    {
        node * victim = FindMin(n->right);
        n->data = victim->data;
        Remove(&n->right, n->data);
        UpdateHeight(n);
        BalanceLeft(pPtr);
    }
    else
    {
        node * old = n;
        *pPtr = n->left?n->left:n->right;
        delete old;
    }
}
```
### Insert函数精简

上文提到过，Insert函数判断失衡类型时有两种方式，如果我们采用第二种，即判断当前节点左右子树的高度差是否大于2，那么这一流程将与Remove函数的判断失衡并旋转的代码一模一样。由此，我们可以在Insert函数中也调用BalanceRight和BalanceLeft函数。
```c++
void Insert(node ** pPtr, int data)
{
    node * n = *pPtr;
    if(!n)
        *pPtr = new node(data);
    else if(data < n->data)
    {
        Insert(&n->left, data);
        BalanceLeft(pPtr);
    }
    else if(n->data < data)
    {
        Insert(&n->right, data);
        BalanceRight(pPtr);
    }
    UpdateHeight(*pPtr);
}
```

## 完整代码附录
### 以Height作为平衡条件

```c++
struct node{
    int data;
    int height;
    node * left, * right;
    node(int data):data(data), height(0), left(nullptr), right(nullptr){}
};

static int Height(node * root)
{
    return root?root->height:0;
}

static void UpdateHeight(node * root)
{
    root->height = max(Height(root->left), Height(root->right))+1;
}

static void RotateR(node ** pRoot)
{
    node * root = *pRoot;
    node * pivot = (*pRoot)->left;

    root->left = pivot->right;
    pivot->right = root;
    *pRoot = pivot;
    UpdateHeight(root);
    UpdateHeight(pivot);
}

static void RotateL(node ** pRoot)
{
    node * root = *pRoot;
    node * pivot = (*pRoot)->right;

    root->right = pivot->left;
    pivot->left =  root;
    *pRoot = pivot;
    UpdateHeight(root);
    UpdateHeight(pivot);
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

static node* FindMin(node * root)
{
    node * n = root;
    while(n->left)
        n = n->left;
    return n;
}

void Insert(node ** pPtr, int data)
{
    node * n = *pPtr;
    if(!n)
        *pPtr = new node(data);
    else if(data < n->data)
    {
        Insert(&n->left, data);
        if(Height(n->left)-Height(n->right)>=2)
        {
            //if(Height(n->left->left)>=Height(n->left->right))
            if(data < n->left->data)
                RotateR(pPtr);
            else
                RotateLR(pPtr);
        }
    }
    else if(n->data < data)
    {
        Insert(&n->right, data);
        if(Height(n->right)-Height(n->left)>=2)
        {
            //if(Height(n->right->right)>=Height(n->right->left))
            if(n->right->data < data)
                RotateL(pPtr);
            else
                RotateRL(pPtr);
        }
    }
    UpdateHeight(*pPtr);
}

void Remove(node ** pRoot, int data)
{
    node * n = *pRoot;
    if(!n)
        return;
    if(data < n->data)
    {
        Remove(&n->left, data);
        UpdateHeight(n);
        if(Height(n->right)-Height(n->left) == 2)
        {
            if(Height(n->right->right) >= Height(n->right->left))
                RotateL(pRoot);
            else
                RotateRL(pRoot);
        }
    }
    else if(n->data < data)
    {
        Remove(&n->right, data);
        UpdateHeight(n);
        if(Height(n->left)-Height(n->right)==2)
        {
            if(Height(n->left->left)>=Height(n->left->right))
                RotateR(pRoot);
            else
                RotateLR(pRoot);
        }
    }
    else
    {
        if(n->left && n->right)
        {
            node * victim = FindMin(n->right);
            n->data = victim->data;
            Remove(&n->right, n->data);
            UpdateHeight(n);
            if(Height(n->left)-Height(n->right)==2)
            {
                if(Height(n->left->left)>=Height(n->left->right))
                    RotateR(pRoot);
                else
                    RotateLR(pRoot);
            }
        }
        else
        {
            node * old = n;
            *pRoot = n->left?n->left:n->right;
            delete old;
        }
    }
}
```

将一些重复操作合并精简后：

```c++
struct node{
    int data;
    int height;
    node * left, * right;
    node(int data):data(data), height(0), left(nullptr), right(nullptr){}
};

static int Height(node * root)
{
    return root?root->height:0;
}

static void UpdateHeight(node * root)
{
    root->height = max(Height(root->left), Height(root->right))+1;
}

static void RotateR(node ** pRoot)
{
    node * root = *pRoot;
    node * pivot = (*pRoot)->left;

    root->left = pivot->right;
    pivot->right = root;
    *pRoot = pivot;
    UpdateHeight(root);
    UpdateHeight(pivot);
}

static void RotateL(node ** pRoot)
{
    node * root = *pRoot;
    node * pivot = (*pRoot)->right;

    root->right = pivot->left;
    pivot->left =  root;
    *pRoot = pivot;
    UpdateHeight(root);
    UpdateHeight(pivot);
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

static node* FindMin(node * root)
{
    node * n = root;
    while(n->left)
        n = n->left;
    return n;
}

static void BalanceLeft(node ** pPtr)
{
    node * n = *pPtr;
    if(Height(n->left)-Height(n->right)>=2)
    {
        if(Height(n->left->left)>=Height(n->left->right))
            RotateR(pPtr);
        else
            RotateLR(pPtr);
    }
}

static void BalanceRight(node ** pPtr)
{
    node * n = *pPtr;
    if(Height(n->right)-Height(n->left)>=2)
    {
        if(Height(n->right->right)>=Height(n->right->left))
            RotateL(pPtr);
        else
            RotateRL(pPtr);
    }
}

void Insert(node ** pPtr, int data)
{
    node * n = *pPtr;
    if(!n)
        *pPtr = new node(data);
    else if(data < n->data)
    {
        Insert(&n->left, data);
        BalanceLeft(pPtr);
    }
    else if(n->data < data)
    {
        Insert(&n->right, data);
        BalanceRight(pPtr);
    }
    UpdateHeight(*pPtr);
}

void Remove(node ** pPtr, int data)
{
    node * n = *pPtr;
    if(!n)
        return;
    if(data < n->data)
    {
        Remove(&n->left, data);
        UpdateHeight(n);
        BalanceRight(pPtr);
    }
    else if(n->data < data)
    {
        Remove(&n->right, data);
        UpdateHeight(n);
        BalanceLeft(pPtr);
    }
    else if(n->left && n->right)
    {
        node * victim = FindMin(n->right);
        n->data = victim->data;
        Remove(&n->right, n->data);
        UpdateHeight(n);
        BalanceLeft(pPtr);
    }
    else
    {
        node * old = n;
        *pPtr = n->left?n->left:n->right;
        delete old;
    }
}
```
### 以bf作为平衡判断条件  
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

### 判断是否为AVL树

```c++
static bool IsBalancedImpl(node* root, int & h)
{
    int h1, h2;
    h1 = h2 = 0;
    if (root->left && !IsBalancedImpl(root->left, h1))
        return false;
    if (root->right && !IsBalancedImpl(root->right, h2))
        return false;
    h = max(h1, h2) + 1;
    assert(root->bf == (h1 - h2));
    return abs(h1 - h2) <= 1;
}

static bool IsBalanced(node* root) {
    if (!root)
        return true;
    int h = 0;
    return IsBalancedImpl(root, h);
}

static bool IsValidBSTImpl(node* root, int & last, bool & lastValid)
{
    if (root->left && !IsValidBSTImpl(root->left, last, lastValid))
        return false;
    if (lastValid && last >= root->data)
        return false;
    last = root->data;
    lastValid = true;
    if (root->right && !IsValidBSTImpl(root->right, last, lastValid))
        return false;
    return true;
}

static bool IsValidBST(node* root) {
    if (!root)
        return true;
    int last;
    bool lastValid = false;
    return IsValidBSTImpl(root, last, lastValid);
}

bool IsBBST(node * root)
{
    return IsBalanced(root) && IsValidBST(root);
}
```
## 参考链接
[github上某源码](https://github.com/i-square/Data-Structure/blob/master/Chapter04/AVLTree.h)  