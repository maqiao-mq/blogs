```c++
#define M 3

struct node
{
	int keyNum;
	node *parent;
	int keys[M];
	node *nodes[M + 1];
	node(node *parent) : keyNum(0), parent(parent), keys(), nodes() {}
};

struct Result
{
	node *ptr;
	int index;
	bool status;
	Result(node *ptr, int index, bool status) : ptr(ptr), index(index), status(status) {}
};

struct Position
{
	node *ptr;
	int nodeIndex;
	Position(node *ptr, int nodeIndex) : ptr(ptr), nodeIndex(nodeIndex) {}
};

using SearchPath = stack<Position>;

static int SearchInsideNode(node *n, int key)
{
	int l = 0, h = n->keyNum - 1;
	while (l <= h)
	{
		int m = l + (h - l) / 2;
		if (n->keys[m] < key)
		{
			l = m + 1;
		}
		else
		{
			h = m - 1;
		}
	}
	return l;
}

Result Search(node *root, int key)
{
	node *cur = root;
	node *parent = NULL;
	bool found = false;
	int index = 0;

	while (cur && !found)
	{
		index = SearchInsideNode(cur, key);
		if (index == cur->keyNum || key < cur->keys[index])
		{
			parent = cur;
			cur = cur->nodes[index];
		}
		else
			found = true;
	}
	if (found)
		return {cur, index, true};
	return {parent, index, false};
}

static Result SearchForInsert(node *root, int key)
{
	return Search(root, key);
}

static void InsertIntoNode(node *n, int index, int key, node *child, bool isChildGreatThanKey)
{
	memmove(&n->keys[index + 1], &n->keys[index], (n->keyNum - index) * sizeof(n->keys[0]));
	memmove(&n->nodes[index + 1 + isChildGreatThanKey], &n->nodes[index + isChildGreatThanKey],
			(n->keyNum - index + !isChildGreatThanKey) * sizeof(node *));
	n->keys[index] = key;
	n->nodes[index + isChildGreatThanKey] = child;
	n->keyNum++;
	if (child)
		child->parent = n;
}

static void NewRoot(node **pRoot, int key, node *right)
{
	node *root = new node(nullptr);
	node *left = *pRoot;
	root->keys[0] = key;
	root->nodes[0] = left;
	root->nodes[1] = right;
	if (left)
		left->parent = root;
	if (right)
		right->parent = root;
	root->keyNum++;
	*pRoot = root;
}

static void NodeSplit(node *cur, node **newNode)
{
	node *right = new node(cur->parent);
	int m = M / 2;
	for (int i = m + 1; i <= M; ++i)
	{
		if (cur->nodes[i])
			cur->nodes[i]->parent = right;
	}
	right->keyNum = M - m - 1;
	memcpy(right->keys, &cur->keys[m + 1], sizeof(cur->keys[0]) * (M - m - 1));
	memcpy(right->nodes, &cur->nodes[m + 1], sizeof(node *) * (M - m));
	cur->keyNum = m;
	*newNode = right;
}

bool Insert(node **pRoot, int key)
{
	Result res = Search(*pRoot, key);
	if (res.status)
		return false;

	node *cur = res.ptr;
	int curKey = key;
	node *right = nullptr;
	bool finished = false;
	int index = res.index;
	while (cur && !finished)
	{
		InsertIntoNode(cur, index, curKey, right, true);
		if (cur->keyNum < M)
			finished = true;
		else
		{
			curKey = cur->keys[M / 2];
			NodeSplit(cur, &right);
			cur = cur->parent;
			if (cur)
				index = SearchInsideNode(cur, curKey);
		}
	}
	if (!finished)
		NewRoot(pRoot, curKey, right);
	return true;
}

Result SearchForRemove(node *root, int key, SearchPath *path)
{
	node *cur = root;
	node *parent = NULL;
	bool found = false;
	int index = 0;

	while (cur && !found)
	{
		index = SearchInsideNode(cur, key);
		if (index == cur->keyNum || key < cur->keys[index])
		{
			path->push({cur, index});
			parent = cur;
			cur = cur->nodes[index];
		}
		else
		{
			path->push({cur, index + 1});
			found = true;
		}
	}
	if (found)
		return {cur, index, true};
	return {parent, index, false};
}

static bool FindMin(node *cur, SearchPath *path)
{
	if (!cur)
		return false;
	while (cur)
	{
		path->push({cur, 0});
		cur = cur->nodes[0];
	}
	return true;
}

static void EraseFromNode(node *n, int keyIndex, bool isChildGreatThanKey)
{
	memmove(&n->keys[keyIndex], &n->keys[keyIndex + 1], (n->keyNum - keyIndex - 1) * sizeof(n->keys[0]));
	memmove(&n->nodes[keyIndex + isChildGreatThanKey], &n->nodes[keyIndex + 1 + isChildGreatThanKey], (n->keyNum - keyIndex - isChildGreatThanKey) * sizeof(node *));
	n->keyNum--;
}

static void GetSibling(node *parent, int nodeParentIndex, node **left, node **right)
{
	if (!parent)
	{
		*left = nullptr;
		*right = nullptr;
		return;
	}
	*left = nodeParentIndex == 0 ? nullptr : parent->nodes[nodeParentIndex - 1];
	*right = nodeParentIndex == parent->keyNum ? nullptr : parent->nodes[nodeParentIndex + 1];
}

static void BorrowFromRightSibling(node *cur, node *sibling, node *parent, int keyParentIndex)
{
	int pivot = sibling->keys[0];
	node *siblingChild = sibling->nodes[0];
	EraseFromNode(sibling, 0, false);
	InsertIntoNode(cur, cur->keyNum, parent->keys[keyParentIndex], siblingChild, true);
	parent->keys[keyParentIndex] = pivot;
}

static void BorrowFromLeftSibling(node *cur, node *sibling, node *parent, int keyParentIndex)
{
	int pivot = sibling->keys[sibling->keyNum - 1];
	node *siblingChild = sibling->nodes[sibling->keyNum];
	EraseFromNode(sibling, sibling->keyNum - 1, true);
	InsertIntoNode(cur, 0, parent->keys[keyParentIndex], siblingChild, false);
	parent->keys[keyParentIndex] = pivot;
}

static void MergeSibling(node *left, node *right, node *parent, int keyParentIndex)
{
	left->keys[left->keyNum++] = parent->keys[keyParentIndex];
	for (int i = 0; i < right->keyNum + 1; ++i)
	{
		if (right->nodes[i])
			right->nodes[i]->parent = left;
	}
	memcpy(&left->keys[left->keyNum], &right->keys[0], sizeof(right->keys[0]) * right->keyNum);
	memcpy(&left->nodes[left->keyNum], &right->nodes[0], sizeof(node *) * (right->keyNum + 1));
	left->keyNum += right->keyNum;
	delete right;
	EraseFromNode(parent, keyParentIndex, true);
}

static inline int PrevKeyIndexOfNode(int nodeIndex)
{
	return nodeIndex == 0 ? 0 : nodeIndex - 1;
}

bool Remove(node **pRoot, int key)
{
	SearchPath path;
	Result res = SearchForRemove(*pRoot, key, &path);
	node *cur;
	int index;

	if (!res.status)
		return false;
	bool ret = FindMin(res.ptr->nodes[res.index + 1], &path);
	cur = path.top().ptr;
	index = path.top().nodeIndex;
	if (ret)
	{
		res.ptr->keys[res.index] = cur->keys[PrevKeyIndexOfNode(index)];
	}
	EraseFromNode(cur, PrevKeyIndexOfNode(index), true);

	do
	{
		cur = path.top().ptr;
		index = path.top().nodeIndex;
		path.pop();
		if (cur->keyNum >= M / 2)
			break;
		else
		{
			node *left, *right, *parent;
			int parentIndex;
			if (path.empty())
				break;
			parent = path.top().ptr;
			parentIndex = path.top().nodeIndex;
			GetSibling(parent, parentIndex, &left, &right);
			if (right && right->keyNum > M / 2)
			{
				BorrowFromRightSibling(cur, right, parent, parentIndex);
				break;
			}
			if (left && left->keyNum > M / 2)
			{
				BorrowFromLeftSibling(cur, left, parent, parentIndex - 1);
				break;
			}
			if (right)
				MergeSibling(cur, right, parent, parentIndex);
			else if (left)
				MergeSibling(left, cur, parent, parentIndex - 1);
		}
	} while (!path.empty());
	node *root = *pRoot;
	if (root->keyNum == 0)
	{
		*pRoot = root->nodes[0];
		delete root;
		if (*pRoot)
			(*pRoot)->parent = nullptr;
	}
	return true;
}

bool IsBTreeImpl(node *root, int low, int high, int &height, node *parent)
{
	int childHeight = -1;
	height = 0;
	if (!root)
		return true;
	if (root->parent != parent)
		return false;
	int curLow = low;
	int curChildHeight = 0;
	if (root->keyNum == 0)
		return false;
	for (int i = 0; i < root->keyNum; ++i)
	{
		int curKey = root->keys[i];
		if (curKey <= low || curKey >= high)
			return false;
		if (!IsBTreeImpl(root->nodes[i], curLow, curKey, curChildHeight, root))
			return false;
		curLow = curKey;
		if (childHeight == -1)
			childHeight = curChildHeight;
		else if (childHeight != curChildHeight)
			return false;
	}
	if (!IsBTreeImpl(root->nodes[root->keyNum], curLow, high, curChildHeight, root))
		return false;
	if (childHeight != curChildHeight)
		return false;
	height = childHeight + 1;
	return true;
}

bool IsBTree(node *root)
{
	int height = 0;
	return IsBTreeImpl(root, INT_MIN, INT_MAX, height, NULL);
}

int main()
{
	int N = 10000;
	vector<int> v(N);
	iota(v.begin(), v.end(), 0);
	std::random_device rd;
	std::mt19937 g(rd());
	shuffle(v.begin(), v.end(), g);
	node *root = nullptr;
	for (int i = 0; i < N; ++i)
	{
		int val = v[i];
		assert(Insert(&root, val));
		assert(IsBTree(root));
	}

	shuffle(v.begin(), v.end(), g);
	for (int i = 0; i < v.size(); ++i)
	{
		int val = v[i];
		assert(Remove(&root, val));
		assert(IsBTree(root));
		assert(!Search(root, val).status);
	}
	assert(!root);
	return 0;
}
```

