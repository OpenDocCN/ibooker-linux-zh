# 第三章：BPF 地图

通过消息传递来调用程序中的行为是软件工程中广泛使用的技术。程序可以通过发送消息来修改另一个程序的行为；这也允许这些程序之间交换信息。关于 BPF 最迷人的一个方面是，运行在内核上的代码和加载了该代码的程序可以在运行时使用消息传递来彼此通信。

在本章中，我们介绍了 BPF 程序和用户空间程序如何进行交互。我们描述了内核与用户空间之间的不同通信渠道，以及它们如何存储信息。我们还展示了这些通道的用例以及如何使这些通道中的数据在程序初始化之间保持持久性。

BPF 地图是驻留在内核中的键/值存储。任何了解它们的 BPF 程序都可以访问这些地图。运行在用户空间的程序也可以使用文件描述符访问这些地图。您可以在地图中存储任何类型的数据，只要您事先正确指定数据大小。内核将键和值视为二进制数据块，并不关心您在地图中保存了什么。

BPF 验证器包含多个保障措施，以确保您创建和访问地图的方式是安全的。当我们解释如何访问这些地图中的数据时，我们将讨论这些保障措施。

# 创建 BPF 地图

创建 BPF 地图的最直接方法是使用 `bpf` 系统调用。当调用中的第一个参数是 `BPF_MAP_CREATE` 时，您告诉内核您要创建一个新的地图。此调用将返回与您刚刚创建的地图关联的文件描述符标识符。系统调用中的第二个参数是此地图的配置：

```
union bpf_attr {
  struct {
    __u32 map_type;     /* one of the values from bpf_map_type */
    __u32 key_size;     /* size of the keys, in bytes */
    __u32 value_size;   /* size of the values, in bytes */
    __u32 max_entries;  /* maximum number of entries in the map */
    __u32 map_flags;    /* flags to modify how we create the map */
  };
}
```

系统调用中的第三个参数是此配置属性的大小。

例如，您可以创建一个哈希表地图，以以下方式存储无符号整数作为键和值：

```
union bpf_attr my_map {
  .map_type = BPF_MAP_TYPE_HASH,
  .key_size = sizeof(int),
  .value_size = sizeof(int),
  .max_entries = 100,
  .map_flags = BPF_F_NO_PREALLOC,
};

int fd = bpf(BPF_MAP_CREATE, &my_map, sizeof(my_map));
```

如果调用失败，内核将返回值 `-1`。它失败的原因可能有三种。如果其中一个属性无效，内核将将 `errno` 变量设置为 `EINVAL`。如果执行操作的用户权限不足，内核将将 `errno` 变量设置为 `EPERM`。最后，如果没有足够的内存来存储地图，内核将将 `errno` 变量设置为 `ENOMEM`。

在接下来的几节中，我们将引导您通过不同的示例，展示如何使用 BPF 地图执行更高级的操作；让我们从创建任何类型的地图的更直接方式开始。

## ELF 约定创建 BPF 地图

内核包含了几个约定和帮助函数，用于生成和使用 BPF 地图。你可能会发现这些约定比直接的系统调用执行更常见，因为它们更易读、更易于跟随。请记住，即使在内核中直接运行时，这些约定仍然使用`bpf`系统调用来创建地图，如果你事先不知道需要哪种地图，直接使用系统调用会更有用。

辅助函数`bpf_map_create`包装了你刚才看到的代码，使得按需初始化地图变得更容易。我们可以用一行代码创建之前的地图：

```
int fd;
fd = bpf_create_map(BPF_MAP_TYPE_HASH, sizeof(int), sizeof(int), 100,
    BPF_F_NO_PREALOC);
```

如果你知道程序中需要哪种地图，你也可以预定义它。这对于提前了解程序使用的地图更有帮助：

```
struct bpf_map_def SEC("maps") my_map = {
      .type        = BPF_MAP_TYPE_HASH,
      .key_size    = sizeof(int),
      .value_size  = sizeof(int),
      .max_entries = 100,
      .map_flags   = BPF_F_NO_PREALLOC,
};
```

当你以这种方式定义地图时，你正在使用所谓的*节属性*，在这种情况下是`SEC("maps")`。这个宏告诉内核这个结构是一个 BPF 地图，并且应该相应地创建它。

你可能注意到，在这个新例子中我们没有与地图关联的文件描述符标识符。在这种情况下，内核使用一个叫做`map_data`的全局变量来存储程序中地图的信息。这个变量是一个结构体数组，按照你在代码中指定的每个地图的顺序排列。例如，如果前一个地图是你代码中指定的第一个地图，你可以从数组的第一个元素中获取文件描述符标识符：

```
fd = map_data[0].fd;
```

你也可以通过这个结构访问地图的名称及其定义；这些信息有时对调试和跟踪非常有用。

初始化地图后，你可以开始在内核和用户空间之间发送消息。现在让我们看看如何处理这些地图存储的数据。

# 与 BPF 地图的工作

内核和用户空间之间的通信将是你编写的每个 BPF 程序的一个基本组成部分。访问地图的 API 在编写内核代码和编写用户空间程序代码时有所不同。本节介绍了每种实现的语义和具体细节。

## 更新 BPF 地图中的元素

在创建任何映射之后，您可能希望用信息填充它。内核助手为此目的提供了函数`bpf_map_update_elem`。如果您从内核中运行的程序加载它，该函数的签名与如果您从用户空间中运行的程序加载它时稍有不同，因为当您在内核中工作时，您可以直接访问映射，但是当您在用户空间工作时，您将使用文件描述符引用它们。其行为也略有不同。在内核中运行的代码可以直接访问内存中的映射，并且可以原子地就地更新元素。然而，在用户空间运行的代码必须向内核发送消息，然后再更新映射之前复制提供的值，这使得更新操作不是原子的。当操作成功时，该函数返回`0`，当操作失败时返回负数。在失败的情况下，全局变量`errno`将填充失败的原因。我们稍后在本章中列出更多与上下文相关的失败案例。

内核中的`bpf_map_update_elem`函数接受四个参数。第一个是我们已经定义的映射的指针。第二个是指向我们想要更新的键的指针。因为内核不知道我们正在更新的键的类型，所以此方法被定义为对`void`的不透明指针，这意味着我们可以传递任何数据。第三个参数是我们想要插入的值。该参数使用与键参数相同的语义。我们在本书中展示了如何利用不透明指针的一些高级示例。您可以使用此函数的第四个参数来更改更新映射的方式。此参数可以取三个值：

+   如果你传递`0`，你告诉内核你希望更新元素（如果存在），或者如果元素不存在则创建该元素。

+   如果你传递`1`，你告诉内核仅在元素不存在时创建该元素。

+   如果你传递`2`，内核将仅在元素存在时更新该元素。

这些值被定义为常量，你也可以使用它们，而不必记住整数语义。这些值分别是`BPF_ANY`代表`0`，`BPF_NOEXIST`代表`1`，以及`BPF_EXIST`代表`2`。

让我们使用在前一节中定义的映射来写一些示例。在我们的第一个示例中，我们向映射中添加一个新值。因为映射是空的，我们可以假设任何更新行为对我们都是有利的。

```
int key, value, result;
key = 1, value = 1234;

result = bpf_map_update_elem(&my_map, &key, &value, BPF_ANY);
if (result == 0)
  printf("Map updated with new element\n");
else
  printf("Failed to update map with new value: %d (%s)\n",
      result, strerror(errno));
```

在这个例子中，我们使用`strerror`来描述`errno`变量中设置的错误。你可以在手册页面上使用`man strerror`了解更多关于这个函数的信息。

现在让我们看看当我们尝试使用相同的键创建一个元素时我们会得到什么结果。

```
int key, value, result;
key = 1, value = 5678;

result = bpf_map_update_elem(&my_map, &key, &value, BPF_NOEXIST);
if (result == 0)
  printf("Map updated with new element\n");
else
  printf("Failed to update map with new value: %d (%s)\n",
      result, strerror(errno));
```

因为我们在映射中已经创建了键为`1`的元素，调用`bpf_map_update_elem`的结果将是`-1`，而`errno`值将是`EEXIST`。这个程序会在屏幕上打印以下内容：

`Failed to update map with new value: -1 (File exists)`

类似地，让我们修改这个程序，尝试更新一个尚不存在的元素：

```
int key, value, result;
key = 1234, value = 5678;

result = bpf_map_update_elem(&my_map, &key, &value, BPF_EXIST);
if (result == 0)
  printf("Map updated with new element\n");
else
  printf("Failed to update map with new value: %d (%s)\n",
      result, strerror(errno));
```

使用`BPF_EXIST`标志，这个操作的结果将再次是`-1`。内核将会将`errno`变量设置为`ENOENT`，程序会打印以下内容：

`Failed to update map with new value: -1 (No such file or directory)`

这些示例展示了如何从内核程序内更新映射。您也可以从用户空间程序内更新映射。执行此操作的助手函数与我们刚刚看到的类似；唯一的区别在于它们使用文件描述符来访问映射，而不是直接使用指向映射的指针。正如您记得的那样，用户空间程序总是使用文件描述符访问映射。因此，在我们的示例中，我们将参数`my_map`替换为全局文件描述符标识符`map_data[0].fd`。在这种情况下，原始代码看起来是这样的：

```
int key, value, result;
key = 1, value = 1234;

result = bpf_map_update_elem(map_data[0].fd, &key, &value, BPF_ANY);
if (result == 0)
  printf("Map updated with new element\n");
else
  printf("Failed to update map with new value: %d (%s)\n",
      result, strerror(errno));
```

尽管您可以在映射中存储的信息类型与您正在使用的映射类型直接相关，但用于填充信息的方法将保持不变，就像您在前面的示例中看到的那样。我们稍后将讨论每种映射类型接受的键和值类型；首先让我们看看如何操作存储数据。

## 从 BPF 映射中读取元素

现在我们已经使用新元素填充了我们的映射，我们可以从我们代码的其他点开始读取它们。在学习了`bpf_map_update_element`之后，阅读 API 会变得很熟悉。

BPF 还提供了两种不同的助手函数来从映射中读取，具体取决于您的代码运行在哪里。这两个助手函数都称为`bpf_map_lookup_elem`。与更新助手函数类似，它们在第一个参数上有所不同；内核方法接受映射的引用，而用户空间助手函数则以映射的文件描述符标识符作为其第一个参数。这两种方法都返回一个整数，表示操作成功或失败，就像更新助手函数一样。这些助手函数的第三个参数是指向您代码中将要存储从映射中读取的值的变量的指针。我们基于您在前一节中看到的代码提供了两个示例。

第一个例子读取了在 BPF 程序在内核上运行时插入映射中的值：

```
int key, value, result; // value is going to store the expected element's value
key = 1;

result = bpf_map_lookup_elem(&my_map, &key, &value);
if (result == 0)
  printf("Value read from the map: '%d'\n", value);
else
  printf("Failed to read value from the map: %d (%s)\n",
      result, strerror(errno));
```

如果我们试图读取的键，`bpf_map_lookup_elem`，返回了一个负数，它将会在`errno`变量中设置错误。例如，如果我们在尝试读取之前没有插入该值，内核将会返回“未找到”错误`ENOENT`。

这个第二个例子与你刚刚看到的例子类似，但这次我们是从在用户空间运行的程序中读取映射：

```
int key, value, result; // value is going to store the expected element's value
key = 1;

result = bpf_map_lookup_elem(map_data[0].fd, &key, &value);
if (result == 0)
  printf("Value read from the map: '%d'\n", value);
else
  printf("Failed to read value from the map: %d (%s)\n",
      result, strerror(errno));
```

如你所见，我们已将`bpf_map_lookup_elem`中的第一个参数替换为映射的文件描述符标识符。助手的行为与前面的示例相同。

这就是我们需要访问 BPF 映射中信息的全部内容。我们将在后面的章节中详细讨论不同工具包如何简化数据访问，使其更加简单。

## 从 BPF 映射中移除元素

我们可以在映射上执行的第三个操作是删除元素。与写入和读取元素一样，BPF 为我们提供了两个不同的帮助程序来删除元素，均称为`bpf_map_delete_element`。与之前的示例一样，当您在运行于内核的程序中使用它们时，这些助手使用映射的直接引用，而在运行于用户空间的程序中使用它们时，则使用映射的文件描述符标识符。

第一个示例在内核运行 BPF 程序时删除了映射中插入的值：

```
int key, result;
key = 1;

result = bpf_map_delete_element(&my_map, &key);
if (result == 0)
  printf("Element deleted from the map\n");
else
  printf("Failed to delete element from the map: %d (%s)\n",
      result, strerror(errno));
```

如果您试图删除的元素不存在，内核将返回一个负数。在这种情况下，它还会将`errno`变量填充为“未找到”错误`ENOENT`。

这个第二个示例在用户空间运行 BPF 程序时删除了该值：

```
int key, result;
key = 1;

result = bpf_map_delete_element(map_data[0].fd, &key);
if (result == 0)
  printf("Element deleted from the map\n");
else
  printf("Failed to delete element from the map: %d (%s)\n",
      result, strerror(errno));
```

您可以看到，我们再次更改了第一个参数，使用了文件描述符标识符。其行为将与内核的助手保持一致。

这结束了可以称为 BPF 映射的创建/读取/更新/删除（CRUD）操作的部分。内核公开了一些额外的函数来帮助您进行其他常见操作；我们将在接下来的两个部分中讨论其中的一些。

## 在 BPF 映射中遍历元素

在本节中我们查看的最后一个操作可以帮助您在 BPF 程序中找到任意元素。有时您可能不知道要查找的元素的确切键，或者只是想查看映射中的内容。BPF 为此提供了一个称为`bpf_map_get_next_key`的指令。与您目前看到的帮助程序不同，此指令仅适用于运行在用户空间的程序。

这个助手为您提供了一种确定性的方法来迭代映射中的元素，但其行为比大多数编程语言中的迭代器更不直观。它接受三个参数。第一个是映射的文件描述符标识符，就像您已经看到的其他用户空间助手一样。接下来的两个参数是它变得棘手的地方。根据官方文档，第二个参数 `key` 是您要查找的标识符，第三个参数 `next_key` 是映射中的下一个键。我们更喜欢将第一个参数称为 `lookup_key` —— 几秒钟之后，你就会明白为什么。当您调用此助手时，BPF 会尝试查找具有您传递的查找键的映射中的元素；然后，它将邻接的键设置为 `next_key` 参数的值。因此，如果您想知道在键 `1` 之后出现哪个键，您需要将 `1` 设置为您的查找键；如果映射有一个与此键相邻的键，BPF 将其设置为 `next_key` 参数的值。

在看例子中 `bpf_map_get_next_key` 如何工作之前，让我们向我们的映射中添加几个元素：

```
int new_key, new_value, it;

for (it = 2; it < 6 ; it++) {
  new_key = it;
  new_value = 1234 + it;
  bpf_map_update_elem(map_data[0].fd, &new_key, &new_value, BPF_NOEXIST);
}
```

如果您想打印映射中的所有值，可以使用一个在映射中不存在的查找键调用 `bpf_map_get_next_key`。这会强制 BPF 从映射的开头开始：

```
int next_key, lookup_key;
lookup_key = -1;

while(bpf_map_get_next_key(map_data[0].fd, &lookup_key, &next_key) == 0) {
  printf("The next key in the map is: '%d'\n", next_key);
  lookup_key = next_key;
}
```

这段代码打印出像这样的东西：

```
The next key in the map is: '1'
The next key in the map is: '2'
The next key in the map is: '3'
The next key in the map is: '4'
The next key in the map is: '5'
```

您可以看到，我们将下一个键分配给 `lookup_key`，这样我们就可以继续迭代映射，直到达到结尾。当 `bpf_map_get_next_key` 到达映射的末尾时，返回的值是一个负数，并且设置了 `errno` 变量为 `ENOENT`。这将中止循环的执行。

正如您可以想象的那样，`bpf_map_get_next_key` 可以查找从映射的任意点开始的键；如果您只想获取另一个特定键的下一个键，您不需要从映射的开头开始。

`bpf_map_get_next_key` 可以对您施加的技巧并不止于此；还有另一种行为您需要注意。许多编程语言在迭代元素之前复制映射中的值。如果您的程序中的其他代码决定变更映射，这可以防止未知行为。如果该代码从映射中删除元素，这尤其危险。在使用 `bpf_map_get_next_key` 遍历值时，BPF 不会复制映射中的值。如果程序的其他部分在您循环遍历值时从映射中删除元素，则 `bpf_map_get_next_key` 在尝试查找已删除元素的键的下一个值时将重新开始。让我们通过一个例子来看看这个情况：

```
int next_key, lookup_key;
lookup_key = -1;

while(bpf_map_get_next_key(map_data[0].fd, &lookup_key, &next_key) == 0) {
  printf("The next key in the map is: '%d'\n", next_key);
  if (next_key == 2) {
    printf("Deleting key '2'\n");
    bpf_map_delete_element(map_data[0].fd &next_key);
  }
  lookup_key = next_key;
}
```

此程序打印出下一个输出：

```
The next key in the map is: '1'
The next key in the map is: '2'
Deleteing key '2'
The next key in the map is: '1'
The next key in the map is: '3'
The next key in the map is: '4'
The next key in the map is: '5'
```

当您使用 `bpf_map_get_next_key` 时，请记住这种行为并不是很直观。

因为我们在本章中涵盖的大多数映射类型表现得像数组一样，当您想访问它们存储的信息时，迭代它们将是一个关键操作。但是，在你下面将看到的时候，还有其他访问数据的函数。

## 查找并删除元素

内核提供的另一个有趣函数用于处理映射的是`bpf_map_lookup_and_delete_elem`。此函数在映射中查找给定键并删除该元素。同时，它将元素的值写入变量以供程序使用。当您使用队列和堆栈映射时，这个函数非常方便，我们将在下一节中描述这些映射。但是，并不限于仅在这些类型的映射中使用。让我们看一个如何在我们先前示例中使用该函数的例子：

```
int key, value, result, it;
key = 1;

for (it = 0; it < 2; it++) {
  result = bpf_map_lookup_and_delete_element(map_data[0].fd, &key, &value);
  if (result == 0)
    printf("Value read from the map: '%d'\n", value);
  else
    printf("Failed to read value from the map: %d (%s)\n",
        result, strerror(errno));
}
```

在这个例子中，我们尝试两次从映射中获取相同的元素。在第一次迭代中，此代码将打印映射中元素的值。然而，由于我们使用了`bpf_map_lookup_and_delete_element`，这第一次迭代还将从映射中删除该元素。当循环第二次尝试获取元素时，此代码将失败，并将“未找到”错误`ENOENT`填入`errno`变量中。

直到现在，我们并没有过多关注当并发操作尝试访问 BPF 映射中的同一信息时会发生什么。接下来我们来谈谈这个问题。

## 并发访问映射元素

处理 BPF 映射的一个挑战是许多程序可以并发访问同一映射。这可能会在我们的 BPF 程序中引入竞争条件，并使映射中资源的访问变得不可预测。为了防止竞争条件，BPF 引入了 BPF 自旋锁的概念，允许您在操作映射元素时锁定对其的访问。自旋锁仅适用于数组、哈希和 cgroup 存储映射。

有两个 BPF 辅助函数用于处理自旋锁：`bpf_spin_lock`锁定一个元素，`bpf_spin_unlock`解锁该元素。这些辅助函数与一个结构体一起工作，该结构体充当访问该元素的信号量。当信号量被锁定时，其他程序无法访问元素的值，并且它们会等待信号量被解锁。同时，BPF 自旋锁引入了一个新标志，用户空间程序可以用来改变该锁的状态；该标志称为`BPF_F_LOCK`。

处理自旋锁的第一步是创建我们要锁定访问的元素，然后添加我们的信号量：

```
struct concurrent_element {
  struct bpf_spin_lock semaphore;
  int count;
}
```

我们将在我们的 BPF 映射中存储此结构，并使用元素内的信号量来防止对其的不良访问。现在，我们可以声明将保存这些元素的映射。此映射必须使用 BPF 类型格式（BTF）进行注释，以便验证器知道如何解释结构。类型格式通过向二进制对象添加调试信息，使内核和其他工具对 BPF 数据结构有了更丰富的理解。因为这段代码将在内核中运行，我们可以使用`libbpf`提供的内核宏来注释此并发映射：

```
struct bpf_map_def SEC("maps") concurrent_map = {
      .type        = BPF_MAP_TYPE_HASH,
      .key_size    = sizeof(int),
      .value_size  = sizeof(struct concurrent_element),
      .max_entries = 100,
};

BPF_ANNOTATE_KV_PAIR(concurrent_map, int, struct concurrent_element);
```

在 BPF 程序中，我们可以使用两个锁定辅助函数来保护这些元素，防止竞争条件。即使信号量被锁定，我们的程序也能够安全地修改元素的值：

```
int bpf_program(struct pt_regs *ctx) {
  int key = 0;
  struct concurrent_element init_value = {};
  struct concurrent_element *read_value;

  bpf_map_create_elem(&concurrent_map, &key, &init_value, BPF_NOEXIST);

  read_value = bpf_map_lookup_elem(&concurrent_map, &key);
  bpf_spin_lock(&read_value->semaphore);
  read_value->count += 100;
  bpf_spin_unlock(&read_value->semaphore);
}
```

此示例使用一个新条目初始化我们的并发映射，该条目可以锁定其值的访问。然后，它从映射中获取该值并锁定其信号量，以便它可以保持计数值，防止数据竞争。在使用完值后，它释放锁，以便其他映射可以安全地访问该元素。

从用户空间，我们可以通过使用标志 `BPF_F_LOCK` 在并发映射中持有元素的引用。您可以将此标志与 `bpf_map_update_elem` 和 `bpf_map_lookup_elem_flags` 辅助函数一起使用。这个标志允许您原地更新元素，而不必担心数据竞争。

###### 注意

当更新散列映射和更新数组和 cgroup 存储映射时，`BPF_F_LOCK` 的行为稍有不同。对于后两者，更新发生在原地，并且在执行更新之前，要更新的元素必须已经存在于映射中。在散列映射的情况下，如果元素尚不存在，则程序会锁定映射中元素的桶，并插入一个新元素。

自旋锁并不总是必需的。如果您只是在映射中聚合值，那么您不需要它们。但是，如果您希望在执行多个操作时确保并发程序不会更改映射中的元素，从而保持原子性，它们将非常有用。

在本节中，您已经看到了可以使用 BPF 映射执行的可能操作；但是，到目前为止，我们只使用了一种类型的映射。BPF 包含许多其他映射类型，您可以在不同情况下使用它们。我们将解释 BPF 定义的所有映射类型，并向您展示如何在不同情况下使用它们的具体示例。

# BPF 映射的类型

[Linux 文档](https://oreil.ly/XfoqK) 将映射定义为通用数据结构，您可以在其中存储不同类型的数据。多年来，内核开发人员添加了许多专门的数据结构，这些数据结构在特定用例中更有效。本节探讨了每种映射类型及其用法。

## 散列表映射

散列表映射是添加到 BPF 的第一个通用映射。它们的定义是 `BPF_MAP_TYPE_HASH` 类型。它们的实现和使用与您可能熟悉的其他散列表类似。您可以使用任意大小的键和值；内核会根据需要为您分配和释放它们。当您在散列表映射上使用 `bpf_map_update_elem` 时，内核会原子地替换元素。

散列表映射在查找时被优化得非常快速；它们对于存储频繁读取的结构化数据非常有用。让我们看一个使用它们来跟踪网络 IP 地址及其速率限制的示例程序：

```
#define IPV4_FAMILY 1
struct ip_key {
  union {
    __u32 v4_addr;
    __u8 v6_addr[16];
  };
  __u8 family;
};

struct bpf_map_def SEC("maps") counters = {
      .type        = BPF_MAP_TYPE_HASH,
      .key_size    = sizeof(struct ip_key),
      .value_size  = sizeof(uint64_t),
      .max_entries = 100,
      .map_flags   = BPF_F_NO_PREALLOC
};
```

在这段代码中，我们声明了一个结构化键，并将其用于保存关于 IP 地址的信息。我们定义了我们的程序将用来跟踪速率限制的映射。你可以看到，我们在这个映射中使用 IP 地址作为键。值将是我们的 BPF 程序从特定 IP 地址接收网络数据包的次数。

让我们编写一个小代码片段，在内核中更新这些计数器：

```
uint64_t update_counter(uint32_t ipv4) {
  uint64_t value;
  struct ip_key key = {};
  key.v4_addr = ip4;
  key.family = IPV4_FAMILY;

  bpf_map_lookup_elem(counters, &key, &value);
  (*value) += 1;
}
```

这个函数接收从网络数据包中提取的 IP 地址，并使用我们声明的复合键进行映射查找。在这种情况下，我们假设之前已经用零值初始化了计数器；否则，`bpf_map_lookup_elem` 调用会返回一个负数。

## 数组映射

数组映射是内核添加的第二种 BPF 映射类型。它们使用类型 `BPF_MAP_TYPE_ARRAY` 来定义。当你初始化一个数组映射时，它的所有元素都预先分配在内存中，并设置为它们的零值。因为这些映射由元素切片支持，所以键是数组中的索引，其大小必须正好是四个字节。

使用数组映射的一个缺点是，映射中的元素不能被删除，也不能使数组比它的大小更小。如果尝试在数组映射上使用 `map_delete_elem`，调用将失败，并且你会得到一个 `EINVAL` 错误作为结果。

数组映射通常用于存储可以更改值的信息，但通常在行为上是固定的。人们使用它们来存储具有预定义分配规则的全局变量。由于你不能删除元素，可以假定特定位置的元素始终表示相同的元素。

另一件需要记住的事情是，`map_update_elem` 不像你在哈希表映射中看到的那样是原子的。如果有更新正在进行，同一个程序可以同时从相同位置读取不同的值。如果你在数组映射中存储计数器，可以使用内核的内置函数 `__sync_fetch_and_add` 对映射的值执行原子操作。

## 程序数组映射

程序数组映射是内核添加的第一个专门映射。它们使用类型 `BPF_MAP_TYPE_PROG_ARRAY` 来定义。你可以使用这种类型的映射来存储对 BPF 程序的引用，使用它们的文件描述符标识符。结合辅助函数 `bpf_tail_call` 使用这个映射，可以让你在程序之间跳转，绕过单个 BPF 程序的最大指令限制，并减少实现复杂性。

当你使用这种专用映射时，有几点需要考虑。首先要记住的是，键和值的大小都必须是四字节。第二点要记住的是，当你跳转到一个新程序时，新程序将重用同一内存堆栈，因此你的程序不会消耗所有可用内存。最后，如果尝试跳转到一个不存在于映射中的程序，尾调用将失败，当前程序将继续执行。

让我们深入一个详细的例子，以更好地理解如何使用这种类型的映射：

```
struct bpf_map_def SEC("maps") programs = {
  .type = BPF_MAP_TYPE_PROG_ARRAY,
  .key_size = 4,
  .value_size = 4,
  .max_entries = 1024,
};
```

首先，我们需要声明我们的新程序映射（正如我们前面提到的，键和值的大小始终为四字节）。

```
int key = 1;
struct bpf_insn prog[] = {
  BPF_MOV64_IMM(BPF_REG_0, 0), // assign r0 = 0
  BPF_EXIT_INSN(),  // return r0
};

prog_fd = bpf_prog_load(BPF_PROG_TYPE_KPROBE, prog, sizeof(prog), "GPL");
bpf_map_update_elem(&programs, &key, &prog_fd, BPF_ANY);
```

我们需要声明要跳转到的程序。在本例中，我们编写了一个 BPF 程序，其唯一目的是返回 0。我们使用`bpf_prog_load`将其加载到内核中，然后将其文件描述符标识符添加到我们的程序映射中。

现在我们已经将该程序存储起来，我们可以编写另一个 BPF 程序来跳转到它。BPF 程序只能跳转到同类型的其他程序；在本例中，我们将程序附加到一个 kprobe 跟踪中，就像我们在第二章中看到的那样。

```
SEC("kprobe/seccomp_phase1")
int bpf_kprobe_program(struct pt_regs *ctx) {
  int key = 1;
  /* dispatch into next BPF program */
  bpf_tail_call(ctx, &programs, &key);

  /* fall through when the program descriptor is not in the map */
  char fmt[] = "missing program in prog_array map\n";
  bpf_trace_printk(fmt, sizeof(fmt));
  return 0;
}
```

使用`bpf_tail_call`和`BPF_MAP_TYPE_PROG_ARRAY`，你可以链式调用高达 32 个嵌套调用。这是一个显式的限制，以防止无限循环和内存耗尽。

## 性能事件数组映射

这些类型的映射将`perf_events`数据存储在一个缓冲环中，实时地在 BPF 程序和用户空间程序之间进行通信。它们被定义为`BPF_MAP_TYPE_PERF_EVENT_ARRAY`类型。它们旨在将内核的跟踪工具发出的事件转发到用户空间程序进行进一步处理。这是最有趣的映射类型之一，也是许多可观察性工具的基础，我们将在接下来的章节中讨论。

让我们看一个例子，说明我们如何追踪计算机执行的所有程序。在跳入 BPF 程序代码之前，我们需要声明从内核发送到用户空间的事件结构：

```
struct data_t {
  u32 pid;
  char program_name[16];
};
```

现在，我们需要创建发送事件到用户空间的映射：

```
struct bpf_map_def SEC("maps") events = {
  .type = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
  .key_size = sizeof(int),
  .value_size = sizeof(u32),
  .max_entries = 2,
};
```

在声明了数据类型和映射之后，我们可以创建捕获数据并将其发送到用户空间的 BPF 程序：

```
SEC("kprobe/sys_exec")
int bpf_capture_exec(struct pt_regs *ctx) {
  data_t data;
  // bpf_get_current_pid_tgid returns the current process identifier
  data.pid = bpf_get_current_pid_tgid() >> 32;
  // bpf_get_current_comm loads the current executable name
  bpf_get_current_comm(&data.program_name, sizeof(data.program_name));
  bpf_perf_event_output(ctx, &events, 0, &data, sizeof(data));
  return 0;
}
```

在这个片段中，我们使用`bpf_perf_event_output`将数据追加到映射中。因为这是一个实时缓冲区，你不需要担心映射中元素的键；内核会负责将新元素添加到映射中，并在用户空间程序处理后刷新它。

在第四章中，我们讨论了这些类型映射的更高级用法，并展示了在用户空间处理程序的示例。

## 每 CPU 哈希地图

这种类型的地图是 `BPF_MAP_TYPE_HASH` 的优化版本。这些地图使用类型 `BPF_MAP_TYPE_PERCPU_HASH` 进行定义。当您分配其中一个地图时，每个 CPU 看到自己的地图的隔离版本，这使得高性能的查找和聚合更加高效。如果您的 BPF 程序收集指标并在哈希表地图中进行聚合，则此类型的地图非常有用。

## 每 CPU 数组地图

这种类型的地图也是 `BPF_MAP_TYPE_ARRAY` 的优化版本。它们使用类型 `BPF_MAP_TYPE_PERCPU_ARRAY` 进行定义。就像前面的地图一样，当您分配其中一个地图时，每个 CPU 看到自己的地图的隔离版本，这使得高性能的查找和聚合更加高效。

## 堆栈跟踪地图

这种类型的地图存储了运行过程中的堆栈跟踪。它们使用类型 `BPF_MAP_TYPE_STACK_TRACE` 进行定义。除了这个地图之外，内核开发者还添加了助手 `bpf_get_stackid` 来帮助您填充这个地图的堆栈跟踪。这个助手接受地图作为参数，以及一系列标志，这样您就可以指定是否只想要来自内核、用户空间或两者的跟踪。该助手返回与添加到地图的元素关联的键。

## Cgroup 数组地图

这种类型的地图存储到 cgroup 的引用。Cgroup 数组地图使用类型 `BPF_MAP_TYPE_CGROUP_ARRAY` 进行定义。从本质上讲，它们的行为类似于 `BPF_MAP_TYPE_PROG_ARRAY`，但它们存储指向 cgroup 的文件描述符标识符。

当您希望在控制流量、调试和测试时在 BPF 地图之间共享 cgroup 引用时，这个地图非常有用。让我们看一个如何填充这个地图的示例。我们从地图的定义开始：

```
struct bpf_map_def SEC("maps") cgroups_map = {
  .type = BPF_MAP_TYPE_CGROUP_ARRAY,
  .key_size = sizeof(uint32_t),
  .value_size = sizeof(uint32_t),
  .max_entries = 1,
};
```

我们可以通过打开包含其信息的文件来检索 cgroup 的文件描述符。我们将打开控制 Docker 容器的基本 CPU 分享的 cgroup，并将该 cgroup 存储在我们的地图中：

```
int cgroup_fd, key = 0;
cgroup_fd = open("/sys/fs/cgroup/cpu/docker/cpu.shares", O_RDONLY);

bpf_update_elem(&cgroups_map, &key, &cgroup_fd, 0);
```

## LRU 哈希和每 CPU 哈希地图

这两种类型的地图是哈希表地图，就像您之前看到的那些，但它们还实现了内部 LRU 缓存。LRU 是最近最少使用的缩写，这意味着如果地图已满，这些地图将删除不经常使用的元素，以便为地图中的新元素腾出空间。因此，只要您不介意丢失最近未使用的元素，您可以使用这些地图来插入超出最大限制的元素。它们的类型分别是 `BPF_MAP_TYPE_LRU_HASH` 和 `BPF_MAP_TYPE_LRU_PERCPU_HASH`。

这种地图的 `per cpu` 版本与您之前看到的其他 `per cpu` 地图略有不同。这个地图只保留一个哈希表来存储地图中的所有元素，并且每个 CPU 使用不同的 LRU 缓存，以确保每个 CPU 中使用最频繁的元素仍然保留在地图中。

## LPM Trie 地图

LPM 前缀树地图是一种使用最长前缀匹配（LPM）查找地图中元素的地图类型。LPM 是一种算法，它从树中选择与任何其他匹配中最长查找键匹配的元素。此算法用于路由器和其他设备中，这些设备保持流量转发表以将 IP 地址与特定路由匹配。这些地图使用类型 `BPF_MAP_TYPE_LPM_TRIE` 定义。

这些地图要求其键大小为八的倍数，并且在 8 到 2048 的范围内。如果您不想实现自己的键，内核提供了一个名为 `bpf_lpm_trie_key` 的结构体，您可以用来创建这些键。

在下一个示例中，我们向地图添加两条转发路由，并尝试将 IP 地址与正确的路由匹配。首先，我们需要创建地图：

```
struct bpf_map_def SEC("maps") routing_map = {
  .type = BPF_MAP_TYPE_LPM_TRIE,
  .key_size = 8,
  .value_size = sizeof(uint64_t),
  .max_entries = 10000,
  .map_flags = BPF_F_NO_PREALLOC,
};
```

我们将使用三条转发路由来填充这个地图：`192.168.0.0/16`、`192.168.0.0/24` 和 `192.168.1.0/24`：

```
uint64_t value_1 = 1;
struct bpf_lpm_trie_key route_1 = {.data = {192, 168, 0, 0}, .prefixlen = 16};
uint64_t value_2 = 2;
struct bpf_lpm_trie_key route_2 = {.data = {192, 168, 0, 0}, .prefixlen = 24};
uint64_t value_3 = 3;
struct bpf_lpm_trie_key route_3 = {.data = {192, 168, 1, 0}, .prefixlen = 24};

bpf_map_update_elem(&routing_map, &route_1, &value_1, BPF_ANY);
bpf_map_update_elem(&routing_map, &route_2, &value_2, BPF_ANY);
bpf_map_update_elem(&routing_map, &route_3, &value_3, BPF_ANY);
```

现在，我们使用相同的关键结构来查找 IP `192.168.1.1/32` 的正确匹配：

```
uint64_t result;
struct bpf_lpm_trie_key lookup = {.data = {192, 168, 1, 1}, .prefixlen = 32};

int ret = bpf_map_lookup_elem(&routing_map, &lookup, &result);
if (ret == 0)
  printf("Value read from the map: '%d'\n", result);
```

在本例中，`192.168.0.0/24` 和 `192.168.1.0/24` 都可以匹配查找 IP，因为它们都在这两个范围内。但是，由于此地图使用 LPM 算法，结果将填充键 `192.168.1.0/24` 的值。

## 地图数组和地图哈希

`BPF_MAP_TYPE_ARRAY_OF_MAPS` 和 `BPF_MAP_TYPE_HASH_OF_MAPS` 是两种存储对其他地图的引用的地图类型。它们仅支持一级间接，因此您不能使用它们来存储地图的地图，以此类推。这确保您不会通过意外存储无限链接地图而消耗所有内存。

当您希望能够在运行时替换整个地图时，这些地图类型非常有用。如果您的所有地图都是全局地图的子级，您可以创建完整状态的快照。内核确保在父地图的任何更新操作等待所有对旧子地图的引用被丢弃之前完成该操作。

## 设备地图映射

这种专门的地图类型存储对网络设备的引用。这些地图使用类型 `BPF_MAP_TYPE_DEVMAP` 定义。它们对于希望在内核级别操作流量的网络应用程序非常有用。您可以建立一个虚拟的端口地图，指向特定的网络设备，然后通过使用辅助 `bpf_redirect_map` 来重定向数据包。

## CPU 地图映射

`BPF_MAP_TYPE_CPUMAP` 是另一种允许您转发网络流量的地图类型。在这种情况下，地图存储了主机中不同 CPU 的引用。与前一种地图类型类似，您可以使用 `bpf_redirect_map` 辅助程序来重定向数据包。然而，这种地图将数据包发送到不同的 CPU。这允许您为可伸缩性和隔离目的分配特定的 CPU 给网络堆栈。

## 打开套接字地图

`BPF_MAP_TYPE_XSKMAP` 是一种存储对打开套接字的引用的地图类型。与前述地图类似，这些地图对于在套接字之间转发数据包非常有用。

## 套接字数组和哈希地图

`BPF_MAP_TYPE_SOCKMAP`和`BPF_MAP_TYPE_SOCKHASH`是两种存储内核中打开套接字引用的专用地图。与之前的地图一样，这种类型的地图与助手`bpf_redirect_map`一起使用，将当前 XDP 程序的套接字缓冲区重定向到不同的套接字。

它们的主要区别在于其中一个使用数组来存储套接字，另一个使用哈希表。使用哈希表的优势在于你可以直接通过其键访问套接字，而无需遍历整个地图来查找。内核中的每个套接字由一个五元组键标识。这些五元组包括建立双向网络连接所需的必要信息。当你使用这种地图的哈希表版本时，可以将此键作为查找键在你的地图中使用。

## Cgroup 存储和每 CPU 存储地图

这两种类型的地图被引入以帮助开发人员处理附加到 cgroup 的 BPF 程序。正如你在第二章中看到的，你可以附加和分离 BPF 程序到控制组，并通过`BPF_PROG_TYPE_CGROUP_SKB`将它们的运行时隔离到特定的 cgroup 中。这两个地图被定义为类型`BPF_MAP_TYPE_CGROUP_STORAGE`和`BPF_MAP_TYPE_PERCPU_CGROUP_STORAGE`。

这些类型的地图从开发者的角度来看类似于哈希表地图。内核提供了一个结构助手来为这个地图生成键，`bpf_cgroup_storage_key`，其中包含关于 cgroup 节点标识符和附加类型的信息。你可以向这个地图添加任何你想要的值；其访问将被限制在附加 cgroup 内部的 BPF 程序中。

这些地图存在两个限制。第一个是你不能从用户空间创建地图中的新元素。内核中的 BPF 程序可以使用`bpf_map_update_elem`创建元素，但如果从用户空间使用此方法且键不存在，`bpf_map_update_elem`将失败，并设置`errno`为`ENOENT`。第二个限制是你不能从此地图中删除元素。`bpf_map_delete_elem`始终失败，并将`errno`设置为`EINVAL`。

正如你之前看到的其他类似地图一样，这两种地图的主要区别在于`BPF_MAP_TYPE_PERCPU_CGROUP_STORAGE`为每个 CPU 保留了一个不同的哈希表。

## 重用端口套接字地图

这种特殊类型的地图存储可以被系统中打开端口的重用的套接字的引用。它们被定义为类型`BPF_MAP_TYPE_REUSEPORT_SOCKARRAY`。这些地图主要与`BPF_PROG_TYPE_SK_REUSEPORT`程序类型一起使用。结合使用，它们让你可以决定如何过滤和处理来自网络设备的传入数据包。例如，你可以决定哪些数据包发送到哪个套接字，即使这两个套接字都附加到同一个端口。

## 队列地图

队列映射使用先进先出（FIFO）的存储方式来保持映射中的元素。它们的类型定义为`BPF_MAP_TYPE_QUEUE`。FIFO 意味着当您从映射中获取元素时，结果将是在映射中存在时间最长的元素。

`bpf`映射助手在这种数据结构中也是以可预测的方式工作的。当您使用`bpf_map_lookup_elem`时，该映射总是查找映射中最旧的元素。当您使用`bpf_map_update_elem`时，该映射总是将元素追加到队列的末尾，因此您需要在获取此元素之前读取映射中的其余元素。您还可以使用助手`bpf_map_lookup_and_delete`以原子方式获取并从映射中删除较旧的元素。该映射不支持助手`bpf_map_delete_elem`和`bpf_map_get_next_key`。如果尝试使用它们，它们将失败并将`errno`变量设置为`EINVAL`。

您还需要记住关于这些类型映射的一些事情，它们不使用映射键进行查找，并且在初始化这些映射时，键大小必须始终为 0。当您将元素推送到这些映射时，键必须是空值。

让我们看一个如何使用这种类型映射的例子：

```
 struct bpf_map_def SEC("maps") queue_map = {
   .type = BPF_MAP_TYPE_QUEUE,
   .key_size = 0,
   .value_size = sizeof(int),
   .max_entries = 100,
   .map_flags = 0,
 };
```

让我们在这个映射中插入几个元素，并以插入它们的相同顺序检索它们：

```
int i;
for (i = 0; i < 5; i++)
  bpf_map_update_elem(&queue_map, NULL, &i, BPF_ANY);

int value;
for (i = 0; i < 5; i++) {
  bpf_map_lookup_and_delete(&queue_map, NULL, &value);
  printf("Value read from the map: '%d'\n", value);
}
```

该程序打印如下内容：

```
Value read from the map: '0'
Value read from the map: '1'
Value read from the map: '2'
Value read from the map: '3'
Value read from the map: '4'
```

如果我们尝试从映射中弹出一个新元素，`bpf_map_lookup_and_delete`将返回一个负数，并且`errno`变量将被设置为`ENOENT`。

## 栈映射

栈映射使用后进先出（LIFO）的存储方式来保持映射中的元素。它们的类型定义为`BPF_MAP_TYPE_STACK`。LIFO 意味着当您从映射中获取元素时，结果将是最近添加到映射中的元素。

`bpf`映射助手在这种数据结构中也是以可预测的方式工作的。当您使用`bpf_map_lookup_elem`时，该映射总是查找映射中最新的元素。当您使用`bpf_map_update_elem`时，该映射总是将元素追加到栈的顶部，因此它是第一个要获取的元素。您还可以使用助手`bpf_map_lookup_and_delete`以原子方式获取并从映射中删除最新的元素。该映射不支持助手`bpf_map_delete_elem`和`bpf_map_get_next_key`。如果尝试使用它们，它们将始终失败，并将`errno`变量设置为`EINVAL`。

让我们看一个如何使用这个映射的例子：

```
struct bpf_map_def SEC("maps") stack_map = {
  .type = BPF_MAP_TYPE_STACK,
  .key_size = 0,
  .value_size = sizeof(int),
  .max_entries = 100,
  .map_flags = 0,
};
```

让我们在这个映射中插入几个元素，并以插入它们的相同顺序检索它们：

```
int i;
for (i = 0; i < 5; i++)
  bpf_map_update_elem(&stack_map, NULL, &i, BPF_ANY);

int value;
for (i = 0; i < 5; i++) {
  bpf_map_lookup_and_delete(&stack_map, NULL, &value);
  printf("Value read from the map: '%d'\n", value);
}
```

该程序打印如下内容：

```
Value read from the map: '4'
Value read from the map: '3'
Value read from the map: '2'
Value read from the map: '1'
Value read from the map: '0'
```

如果我们尝试从映射中弹出一个新元素，`bpf_map_lookup_and_delete`将返回一个负数，并且`errno`变量将被设置为`ENOENT`。

这些都是您可以在 BPF 程序中使用的所有映射类型。您会发现其中一些比其他更有用；这取决于您正在编写的程序类型。在本书中，我们将看到更多的使用示例，这将帮助您巩固刚刚学到的基础知识。

正如我们之前提到的，BPF 映射作为操作系统中的常规文件存储。我们还没有讨论内核用于保存映射和程序的文件系统的具体特性。下一节将指导您了解 BPF 文件系统及其提供的持久性类型。

# BPF 虚拟文件系统

BPF maps 的一个基本特征是，它们基于文件描述符，这意味着当描述符关闭时，该映射及其所保存的所有信息都会消失。最初的 BPF 映射实现专注于短暂的孤立程序，这些程序之间不共享任何信息。在这些情况下，关闭文件描述符时清除所有数据是有意义的。然而，随着内核中更复杂映射和集成的引入，其开发者意识到需要一种方法来保存映射所持有的信息，即使程序终止并关闭映射的文件描述符。Linux 内核版本 4.4 引入了两个新的系统调用，允许从虚拟文件系统中固定和获取映射和 BPF 程序。固定到该文件系统的映射和 BPF 程序将在创建它们的程序终止后仍保留在内存中。本节介绍如何使用这个虚拟文件系统。

BPF 期望找到这个虚拟文件系统的默认目录是*/sys/fs/bpf*。一些 Linux 发行版默认情况下不会挂载此文件系统，因为它们不假设内核支持 BPF。您可以使用`mount`命令自行挂载它：

```
# mount -t bpf /sys/fs/bpf /sys/fs/bpf
```

与任何其他文件层次结构一样，文件系统中的持久性 BPF 对象由路径标识。您可以以任何对程序有意义的方式组织这些路径。例如，如果您希望在程序之间共享包含 IP 信息的特定映射，则可能希望将其存储在*/sys/fs/bpf/shared/ips*中。正如我们之前提到的，您可以在此文件系统中保存两种类型的对象：BPF 映射和完整的 BPF 程序。这两者都由文件描述符标识，因此与它们交互的接口是相同的。这些对象只能通过`bpf`系统调用进行操作。尽管内核提供了高级别的辅助功能来帮助您与它们交互，但您不能像尝试使用`open`系统调用打开这些文件那样操作它们。

`BPF_PIN_FD`是将 BPF 对象保存在此文件系统中的命令。命令成功后，对象将在文件系统中以您指定的路径可见。如果命令失败，则返回一个负数，并设置全局的`errno`变量以表示错误代码。

`BPF_OBJ_GET` 是用于获取已固定到文件系统的 BPF 对象的命令。此命令使用您分配给对象的路径来加载它。当此命令成功时，它返回与对象关联的文件描述符标识符。如果失败，则返回负数，并且全局的 `errno` 变量设置为特定的错误代码。

让我们看一个示例，展示如何利用内核提供的辅助函数在不同的程序中使用这两个命令。

首先，我们将编写一个程序，创建一个映射，用多个元素填充它，并将其保存在文件系统中：

```
static const char * file_path = "/sys/fs/bpf/my_array";

int main(int argc, char **argv) {
  int key, value, fd, added, pinned;

  fd = bpf_create_map(BPF_MAP_TYPE_ARRAY, sizeof(int), sizeof(int), 100, 0); ![1](img/1.png)
  if (fd < 0) {
    printf("Failed to create map: %d (%s)\n", fd, strerror(errno));
    return -1;
  }

  key = 1, value = 1234;
  added = bpf_map_update_elem(fd, &key, &value, BPF_ANY);
  if (added < 0) {
    printf("Failed to update map: %d (%s)\n", added, strerror(errno));
    return -1;
  }

  pinned = bpf_obj_pin(fd, file_path);
  if (pinned < 0) {
    printf("Failed to pin map to the file system: %d (%s)\n",
        pinned, strerror(errno));
    return -1;
  }

  return 0;
}
```

![1](img/#co_bpf_maps_CO1-1)

这段代码的内容应该已经很熟悉了，因为它来自我们之前的示例。首先，我们创建了一个具有一个固定大小元素的哈希表映射。然后我们更新映射以添加该元素。如果尝试添加更多元素，`bpf_map_update_elem` 将会失败，因为这会导致映射溢出。

我们使用辅助函数 `pbf_obj_pin` 将地图保存在文件系统中。在程序终止后，您可以检查您的机器上该路径下是否有新文件：

```
ls -la /sys/fs/bpf
total 0
drwxrwxrwt 2 root  root  0 Nov 24 13:56 .
drwxr-xr-x 9 root  root  0 Nov 24 09:29 ..
-rw------- 1 david david 0 Nov 24 13:56 my_map
```

现在，我们可以编写一个类似的程序，从文件系统加载该映射并打印我们插入的元素。通过这种方式，我们可以验证我们正确保存了映射：

```
static const char * file_path = "/sys/fs/bpf/my_array";

int main(int argc, char **argv) {
  int fd, key, value, result;

  fd = bpf_obj_get(file_path);
  if (fd < 0) {
    printf("Failed to fetch the map: %d (%s)\n", fd, strerror(errno));
    return -1;
  }

  key = 1;
  result = bpf_map_lookup_elem(fd, &key, &value);
  if (result < 0) {
    printf("Failed to read value from the map: %d (%s)\n",
        result, strerror(errno));
    return -1;
  }

  printf("Value read from the map: '%d'\n", value);
  return 0;
```

能够将 BPF 对象保存在文件系统中为更有趣的应用打开了大门。您的数据和程序不再绑定于单个执行线程。信息可以被不同的应用程序共享，并且 BPF 程序甚至可以在创建它们的应用程序终止后继续运行。这为它们提供了一种额外的可用性级别，这是在没有 BPF 文件系统的情况下无法实现的。

# 结论

在内核和用户空间之间建立通信渠道对于充分利用任何 BPF 程序至关重要。在本章中，您学习了如何创建 BPF 映射以建立通信，并学习了如何操作它们。我们还描述了您可以在程序中使用的映射类型。随着您在本书中的进展，您将看到更多具体的映射示例。最后，您学会了如何将整个映射固定到系统中，使它们及其包含的信息在崩溃和中断时依然可用。

BPF 映射是内核和用户空间之间通信的核心总线。在本章中，我们建立了您理解它们所需的基本概念。在下一章中，我们将更广泛地使用这些数据结构来共享数据。我们还将向您介绍更多使与 BPF 映射更高效的附加工具。

在接下来的章节中，你将看到 BPF 程序和映射如何共同工作，从内核的视角为你提供跟踪能力。我们探讨了将程序附加到内核不同入口点的不同方法。最后，我们讨论了如何以一种使应用程序更易于调试和观察的方式表示多个数据点。
