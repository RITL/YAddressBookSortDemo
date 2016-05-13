# YAddressBookSortDemo
根据UILocalizedIndexedCollation对通讯录中的联系人进行分组的Demo

最近写了两篇关于通讯录的博文，通过前两篇博文的简要描述与介绍，基本是能够读出通讯录中的联系人，并能完成对通讯录增删改的操作，但在真实开发中，列出联系人之后是需要分组的，毕竟不能把联系人无规律的排列在<code>tableView</code>上吧。那么这里就顺水推舟，再介绍一下针对联系人分组特别方便的原生类：<code>UILocalizedIndexedCollation</code>
<br><br>
<div align="center"><label>效果图的先后对比</label><div>
<br>
<div align="center">
<img src="http://img.blog.csdn.net/20160513153513328" height="500"></img><img src="http://img.blog.csdn.net/20160513153535615" height="500"></img>
</div>

<br>
<div align="left">
文本承接的博文:<br>
[iOS开发------获取系统联系人(AddressBook篇)](http://blog.csdn.net/runintolove/article/details/51371996)<br>
[iOS开发------操作通讯录(AddressBook篇)&通讯录UI(AddressBookUI篇)](http://blog.csdn.net/runintolove/article/details/51387594)<br>
博文:[iOS开发------通讯录之根据联系人名字分组(UILocalizedIndexedCollation)](http://blog.csdn.net/RunIntoLove/article/details/51395389)

<br>
#UILocalizedIndexedCollation

##初始化
Demo主页控制器中定义的所有属性
```Objective-C
//存放联系人的数组，存放直接请求出的联系人数组
@property (nonatomic, copy)NSArray <YContactObject *> *  contactObjects;

//存放索引的数组，(e.g. A-Z,# in US/English)
@property (nonatomic, copy)NSArray <NSString *> * titles;

//负责进行联系人分组的原生类
@property (nonatomic, strong)UILocalizedIndexedCollation * localizedCollation;

//存放处理过的数组，真正的数据源
@property (nonatomic, copy)NSArray <NSArray *> * handleContactObjects;

//负责请求联系人对象
@property (nonatomic, strong) YContactsManager * contactManager;
```

在ViewDidLoad中进行属性的初始化
```Objective-C
- (void)viewDidLoad {
    [super viewDidLoad];
    
    //初始化属性
    self.contactManager = [YContactsManager shareInstance];
    self.titles = [NSMutableArray arrayWithCapacity:0];
    self.localizedCollation = [UILocalizedIndexedCollation currentCollation];
    
    //开始请求
    [self requestContacts];

}
```

请求的方法:
```Objective-C
//开始请求所有的联系人
- (void)requestContacts
{
    __weak typeof(self) copy_self = self;
    
    //开始请求
    [self.contactManager requestContactsComplete:^(NSArray<YContactObject *> * _Nonnull contacts) {
        
        //开始赋值
        copy_self.contactObjects = contacts;
        copy_self.titles = copy_self.localizedCollation.sectionTitles;
        
        //刷新
        [copy_self.tableView reloadData];
        
    }];
}
```
<br>
##处理数据源
`_contactObjects`这个数组的作用就是存储一些信息，用Swift的话说它就是一个存储属性，真正用到的数据源是`_handleContactObjects`,所以这里就要重写它的`Getter`方法，博文的中心就在一下几行代码中:
```Objective-C
#pragma mark - localizedCollation Setter
-(NSArray<NSArray *> *)handleContactObjects
{
    //初始化数组返回的数组
    NSMutableArray <NSMutableArray *> * contacts = [NSMutableArray arrayWithCapacity:0];
    
    
    /**(注:)
     * 为什么不直接用27，而用count呢，这里取决于初始化方式
     * 初始化方式为[[Class alloc] init],那么这个count = 0
     * 初始化方式为[Class currentCollation],那么这个count = 27
     */
    
    /**** 根据UILocalizedIndexedCollation的27个Title放入27个存储数据的数组 ****/
    for (NSInteger i = 0; i < self.localizedCollation.sectionTitles.count; i++)
    {
        [contacts addObject:[NSMutableArray arrayWithCapacity:0]];
    }
    
    
    //开始遍历联系人对象，进行分组
    for (YContactObject * contactObject in self.contactObjects)
    {
        //获取名字在UILocalizedIndexedCollation标头的索引数
        NSInteger section = [self.localizedCollation sectionForObject:contactObject.nameObject collationStringSelector:@selector(name)];
        
        //根据索引在相应的数组上添加数据
        [contacts[section] addObject:contactObject];
    }
    
    
    //对每个同组的联系人进行排序
    for (NSInteger i = 0; i < self.localizedCollation.sectionTitles.count; i++)
    {
        //获取需要排序的数组
        NSMutableArray * tempMutableArray = contacts[i];
        
        //如果是直接的属性，直接用该方法排序即可，因为楼主自己构建的Model,name是Model中nameObject的属性，所以此方法不能直接用
        //NSArray * sortArray = [self.localizedCollation sortedArrayFromArray:tempMutableArray collationStringSelector:@selector(name)];
        
        
        //这里因为需要通过nameObject的name进行排序，排序器排序(排序方法有好几种，楼主选择的排序器排序)
        NSSortDescriptor * sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"nameObject.name" ascending:true];
        [tempMutableArray sortUsingDescriptors:@[sortDescriptor]];
        contacts[i] = tempMutableArray;
        
    }
    
    //返回
    return [NSArray arrayWithArray:contacts];
}
```

如果对于数组的排序不是很熟，欢迎看一下楼主之前的博文：[ Objective-C学习-数组排序问题](http://blog.csdn.net/runintolove/article/details/48414043)


##实现TableViewDataSource方法

```Objective-C
#pragma mark - Table view data source

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {

    //返回首字母可能的个数
    return self.titles.count;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {

    //根据组数，获取每组联系人的数量
    return self.handleContactObjects[section].count;
}


- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:reuseIdentifier forIndexPath:indexPath];
    
    //fetch Model
    YContactObject * contactObject = [self.handleContactObjects[indexPath.section] objectAtIndex:indexPath.row];
    
    //configture cell..
    cell.textLabel.text = contactObject.nameObject.name;
    cell.detailTextLabel.text = contactObject.phoneObject.firstObject.phoneNumber;
    
    return cell;
}
```
接下来还需要实现下面的三个数据源方法，为什么要实现呢，固定用法? false : false，除了高中老师会这么教( - - 是不是吐槽了会高中老师)，开发文档这么示范的，所以直接复制上的，应该也都知道这几个协议的意义才对.
```Obejctive-C
- (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section
{
    return [[[UILocalizedIndexedCollation currentCollation] sectionTitles] objectAtIndex:section];
}

- (NSArray *)sectionIndexTitlesForTableView:(UITableView *)tableView
{
    return [[UILocalizedIndexedCollation currentCollation] sectionIndexTitles];
}

- (NSInteger)tableView:(UITableView *)tableView sectionForSectionIndexTitle:(NSString *)title atIndex:(NSInteger)index
{
    return [[UILocalizedIndexedCollation currentCollation] sectionForSectionIndexTitleAtIndex:index];
}
```
<br>
##TableViewDelegate方法
为了避免某组没有数据而出现组标题，实现如下代理方法
```Objective-C
#pragma mark - <UITableViewDelegate>
-(CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section
{
    //如果没有数据
    if (self.handleContactObjects[section].count == 0)
    {
        return 0;
    }
    return 15;
}
```
如此通讯录分组就完成了`Thanks()`
