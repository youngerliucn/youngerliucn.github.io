# 通用文件系统的修复机制fsck

​       在使用文件系统作为载体存储文件数据的过程，由于异常掉电或者其他硬件异常错误，总有可能使得文件系统数据不一致，导致文件系统的文件数据无法访问，或者用户访问到不一致的数据，产生更大的损失。

​       因此，为了检查和维护文件系统一致性，绝大部分linux文件系统都提供了fsck工具或类似的工具，便于文件系统存储使用者能够处理和修复文件系统不一致问题。

​       在通用文件系统中，基本都是采用fsck检查和维护文件系统的一致性

## 简介

​       Fsck是linux文件系统提供的检查和修复文件系统一致性的命令工具。

​       在使用fsck工具时，文件系统不应该处于运行状态。

​       fsck工具也不是万能的，当对存放关键数据的文件系统进行修复时，建议先将文件系统整体备份，然后再行修复。

​       fsck会对关键数据进行检查：

## 1.1 superblock检查

如果对superblock了解，可以知道，superblock存储了文件系统的控制信息，包括文件系统的状态、文件系统类型、大小、块大小、块数、索引节点数、操作方法等。

那么，Super block 的一致性检查包括文件系统大小， inode 数量， 空闲的 block 块， 空闲的 inode数量等。文件系统的大小必须大于 super block 和 inode 使用的 block 数的和。文件系统的大小和布局信息是对 fsck 而言至关重要的信息。如果 fsck 在默认的 super block 中的静态的参数中检查到 corrupt，就会通过备份的 super block 来恢复superblock。

 

## 1.2   free block检查

Fsck 会检查所有free blocks，检查 free块的数量与 inode 中声明使用的块的数量的和是否与整个文件系统的所有块数相等。如果在 block allocation maps 中有任何错误， fsck 将根据其计算的 allocated blocks 进行重新组建 block allocation maps。

Super block 中也存有所有 free 块的数量信息， fsck 会把自己检查的结果与 super block 中的信息进行比较。如果这两个数不等，则 fsck 会将检查得到的结果更新到 super block 中。
对文件系统中的
free inode 的数量，也会进行类似的处理。

 

## 1.3   inode state检查

当文件系统中有很多 inode 存在的时候（即很多文件），有可能会有几个 inode corrupt。

文件系统中的 inode table是从 inode2开始顺序被检查的 （inode0标记没有用过的 inode，inode1用来将来的扩展），直到文件系统中的最后一个 inode。Inode 的状态检查包括：format and type, link count, duplicatedblocks, bad blocks, and inode size。

每个 inode 都有一个结构体，描述了 inode 的 type 和 state。 Inode 必须处于六种类型之一：普通 inode， 目录 inode， symbolink inode， special block inode， special character inode，或者是 socket inode。

Indoe 有三种 allocation 状态： unallocated， allocated 和不属于前两种情况的情况。在第三种状态的 inode 就是不正确的 inode，当 inodes 链表被写入坏的数据的时候， inode 有可能进入这种状态。唯一可能修复的方法是 fsck 清空这个 inode（在链表中删除之）。

 

## 1.4   inode links检查

links是计算每一个 inode 与其相连的目录项的数目。 

fsck 从文件系统的 root 目录开始检查每一个 inode 的连接数，并沿着目录树依次查找。每个 inode 的实际 link count 在遍历的时候计算得到。如果存储的 link count 非 0，而计算的 link count 为 0，则此 inode 没有对应的目录项。这种情况下， fsck 将把这个对应的文件放入 lost+fount 目录中。如果存储的 link count 与实际计算所得的值非 0 且不相等，那么可能是 inode 的 link count 在有一个目录加入或删除的时候没有被相应更新。这种情况下， fsck 会用计算得到的值更新存储的值。

每个 inode 都包含一个列表或者是列表的指针，上面记录着这个 inode 所使用的数据块。因为 inode 是这些列表的拥有者，因此，当这些列表存在不一致的情况时，就直接影响到拥有它的 inode。

Fsck 会将一个 inode 声明的 block number 与列表中已经分配的 block number 比较。如果另一个 inode 已经声明了一个 block number，那么这个 block number 就被加入到一个 duplicate block 链表中。否则，已分配的 block list 中会将这个 block number 加入。

对任何 duplicate blocks， fsck 将会遍历 inode list 找到拥有 duplicated block 的 inode。一般而言，拥有最早修改时间的 inode 坏掉的可能性比较大，需要被 clear。如果是这种情况， fsck将执行操作， clear 这两个 inodes。操作必须决定，哪个该留，哪个该 clear。

Fsck 检查每个 inode 声明的 block number 的范围，如果 block number 比文件系统中第一个数据块的块号低，或者比文件系统中的最后一个数据块的块号大，则称为 bad block

 

## 1.5   inode data size检查

每个 inode包含一定数量的 data blocks。实际 data block的数量是所有 allocated data blocks和 indirect blocks 的总和。 Fsck 计算实际的 data blocks 的数量，并与 inode 所记录的数值进行比较。如果两者不一致， fsck 会进行修正。每个 inode 包含了一个 32 位的 size 域。这个数是 inode 对应的文件所含有的字节数。这个 size 域的一致性检查是通过计算与 inode 对应的最大数量的 blocks 数，与实际 inode 保存的数值比较。

 

## 1.6   checking the data with an inode

一个 inode 可以直接或间接的 reference 三种类型的 data blocks。所有的 referenced blocks必须是同种类型。这三种类型是： plain data blocks， symbolic link data blocks 和 directory data blocks。 Plain data blocks 包含文件中保存的信息。 symbolic link data blocks 包含一个 link 中包含的路径名。 Directory data blocks 包含目录项。 Fsck 只能检查 directory data blocks 的有效性。

Fsck 会检查每个 directory data block 的几种一致性： directory inode 指向 unallocated inodes， directory inodes 的数量比文件系统中的 inode 数量大，不正确的“ .”“ ..”directory inode numbers，没有结合在文件系统中的 directories。如果一个 directory data block 中的 inode number references 一个 unallocated inode， fsck 将移除这个 directory entry（目录项）。这种情况只发生在存在硬件异常的情况下。

Fsck 检查与 unallocated blocks（ holes）对应的 directories。这些 directories 应当从不被创建。当发现这些 directory 存在时， fsck 将通过缩短这些 directory 大小到上一个 allocated block结尾的 hole 来提示用户去调整这些 directory。但同时这又意味着另一轮的第一部分检查需要执行---super blockchecking。 Fsck 将提示用户在修改一个含有 unlocated block 的 directory后重新执行 fsck。（ Whenfound, fsck will promptthe user to adjust the length of the offending directory which is done byshortening the size of the directory to the end of the last allocated block preceedingthe hole. Unfortunately, this means that another Phase 1 run has to be done.Fsck will remind the user to rerun fsck after repairing a directory containingan unallocated block.）

如果一个 directory entry 的 inode number reference 不在 inode list 中， fsck 会移除这个directory entry。这一般发生在bad data 被写入到a directory data block 的情况下。 “ .”的 directory inode number entry 必须是 directory data block 的第一个 entry。“ .”的 inode number 必须 reference 它自己。比如，它必须等于 directory data block 的 inode number。“ ..”的 directory inode number entry 必须是 directory data block 的第二个 entry，它的值必须与这个directory entry 的父目录的 inode number 相等（如果是 root directory 的“ ..”，则与其 directory data block 的 inode number 相等）。如果 directory inode numbers 不正确， fsck 将用正确的值取代它。如果目录有许多 hard links，第一个被认为是“ ..”指向的真正的父目录， fsck 会倾向于删除其后出现的名称。（ recommends deletion for the
subsequently discovered names.（不懂这句话）

## 1.7   File system connectivity

Fsck 检查文件系统的 general connectivity。如果目录不被 link 到文件系统中， fsck 会 link directory 到文件系统中的 lost+found directory。这个条件只有在发生硬件异常的时候成立。

