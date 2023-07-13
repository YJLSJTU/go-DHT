# Chord算法实现分布式哈希表

## 论文未提及的算法细节和我的处理方式

- 在结点加入/退出/强制退出时如何处理数据和备份数据？
    - 首先，在我的实现中备份数据由数据所存储结点的后继结点存储。
    - 在新结点加入时，当它在Chord网络中找到后继之先将后继的备份数据清空，然后立刻从后继拷贝它负责的数据并在后继中进行备份，同时将这些数据从后继的数据以及后继的后继的备份数据中删除。这些操作完成之后唯一未正确的部分是新加入结点没有备份其前驱结点的数据，而此备份操作将在`notify()`方法调用时在设置其前驱结点后进行。
    - 在结点退出（非强制）时，在设置状态为离线并关掉其提供远程服务的服务端后“主动通知”前驱和后继结点：让后继调用`update_predecessor()`方法来将其存储的前驱设置为空，并将备份数据转移到数据中同时让其后继进行备份；让前驱调用`stabilize()`方法更新后继列表并让新后继设置自己为前驱并备份数据。
    - 在结点强制退出时，仅设置自身状态为离线并关掉远程服务的服务端，Chord环的及相应的数据及备份存储恢复通过周期性地调用上述方法来达到。
- 如何维护后继列表？
    - 在新节点加入时，找到后继之后将后继作为自己后继列表的第一个元素，如何将该后继的后继列表中的结点依次拷贝到自己后继列表中空缺的位置。
    - 在周期性调用（或结点退出时主动调用）方法`stabilize()`时，将后继列表中第一个元素更新为后继列表中第一个在线的结点（或其在线的位置合法的前驱），并以与上一条同样的方式更新列表中剩余位置的结点。

## 调试时遇到的问题和我的解决方案

- 部分方法（例如`find_successor()`）出现无限递归远程调用。
    - 在递归远程调用方法本身语句执行前特判请求调用对象是否是当前对象。
- 在结点退出/强制退出之后仍然持续不断的被成功远程调用。
    - 我发现当服务端和客户端建立连接之后客户端方应该在服务结束后主动关闭连接，否则即使服务端关闭，已经成功建立的连接仍然允许服务端向客户端进行服务。
- 在`stabilize()`中更新后继列表时将离线后继（第一个在线后继的离线前驱）作为后继列表的第一个结点并用该离线结点的空后继列表更新后继列表，导致后继列表错误并且无在线结点。
    - 在`get_predecessor()`中判断获取到的前驱是否为空，是则返回error使得在调用此方法处获取该信息以避免不必要的错误。
- 因为前驱离线而造成访存数据发生错误。
    - 发现在有一方法的实现中在发现前驱离线后把前驱设置为空，此为多余操作且干扰到在周期性调用方法`update_predecessor()`时将发现的非空离线前驱时的数据转移操作。