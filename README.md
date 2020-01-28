# boltBlockChain(基于golang-blot实现可持续化的区块链)

    完整教程链接如：https://blog.csdn.net/Laughing_G/article/details/104054099

什么是Bolt数据库呢？

Bolt是一个用Go编写的键值数据库。其目标是为了给程序提供一个简单、快捷、稳定的数据库。

安装：

go get github.com/boltdb/bolt/...
基于BOLT的读写、

func main() {
	//打开数据库

	// func Open(path string, mode os.FileMode, options *Options) (*DB, error) {
	db, err := bolt.Open("testDb.db", 0600, nil)//第二个参数为权限
	
	if err != nil {
		fmt.Println(" bolt Open err :", err)
		return
	}

	defer db.Close()

	//创建bucket
    //Updata参数为一个函数类型，是一个事务
	err = db.Update(func(tx *bolt.Tx) error {
		//打开一个bucket
		b1 := tx.Bucket([]byte("bucket1"))

		//没有这个bucket
		if b1 == nil {
			//创建一个bucket
			b1, err = tx.CreateBucket([]byte("bucket1"))
			if err != nil {
				fmt.Printf("tx.CreateBucket err:", err)
				return err
			}

			//写入数据
			b1.Put([]byte("key1"), []byte("hello"))
			b1.Put([]byte("key2"), []byte("world"))

			//读取数据
			v1 := b1.Get([]byte("key1"))
			v2 := b1.Get([]byte("key2"))
			v3 := b1.Get([]byte("key3"))

			fmt.Printf("v1:%s\n", string(v1))
			fmt.Printf("v2:%s\n", string(v2))
			fmt.Printf("v3:%s\n", string(v3))
		}
		return nil
	})

	if err != nil {
		fmt.Printf("db.Update err:", err)
	}

	return
}
只读：

func main() {
	//打开数据库

	// func Open(path string, mode os.FileMode, options *Options) (*DB, error) {
	db, err := bolt.Open("testDb.db", 0400, nil)//第二个参数为权限
	
	if err != nil {
		fmt.Println(" bolt Open err :", err)
		return
	}

	defer db.Close()

	//创建bucket
    //View参数为一个函数类型，是一个事务
	err = db.View(func(tx *bolt.Tx) error {
		//打开一个bucket
		b1 := tx.Bucket([]byte("bucket1"))

		//没有这个bucket
		if b1 == nil {
			return errors.New("bucket do not exist!")
		}
       		v1 := b1.Get([]byte("key1"))
			v2 := b1.Get([]byte("key2"))
			v3 := b1.Get([]byte("key3"))

			fmt.Printf("v1:%s\n", string(v1))
			fmt.Printf("v2:%s\n", string(v2))
			fmt.Printf("v3:%s\n", string(v3))
        
		return nil
	})

	if err != nil {
		fmt.Printf("db.View err:", err)
	}

	return
}


特别注意--血坑，博主在因为此原因排查了一下午（~~）：

切记结构体中的变量要大写：！！！

 结构体中参数首字母没有大写时，别的包虽然可以调用这个结构体，但是找不到这个结构体中没有首字母大写的参数。

附上block.go中修改结构体变量后的代码：

package main

import (
	"bytes"
	"encoding/binary"
	"encoding/gob"
	"fmt"
	"log"
	"time"
)
//1.定义区块结构体

type Block struct {
	//1.版本号
	Version uint64
	//2. 前区块哈希
	PrevHash []byte
	//3. Merkel根（梅克尔根，这就是一个哈希值，我们先不管，我们后面v4再介绍）
	MerkelRoot []byte
	//4. 时间戳
	TimeStamp uint64
	//5. 难度值
	Difficulty uint64
	//6. 随机数，也就是挖矿要找的数据
	Nonce uint64

	//a. 当前区块哈希,正常比特币区块中没有当前区块的哈希，我们为了是方便做了简化！
	Hash []byte
	//b. 数据
	Data []byte
}
//2.创建区块
func NewBlock(data string, prevBlockHash []byte) *Block {
	block := Block{
		Version:    00,
		PrevHash:   prevBlockHash,
		MerkelRoot: []byte{},
		TimeStamp:  uint64(time.Now().Unix()),
		Difficulty: 0, //随便填写的无效值
		Nonce:      0, //同上
		Hash:       []byte{},
		Data:       []byte(data),
	}

	//block.SetHash()
	//创建一个pow对象
	pow := NewProofOfWork(&block)
	//查找随机数，不停的进行哈希运算
	hash, nonce := pow.run()

	//根据挖矿结果对区块数据进行更新（补充）
	block.Hash = hash
	block.Nonce = nonce

	return &block
}

func IntToByte(n uint64)[]byte{
	x:=int64(n)
	bytesBuffer:=bytes.NewBuffer([]byte{})
	err:=binary.Write(bytesBuffer,binary.BigEndian,x)
	if err!=nil{
		fmt.Println(err)
		return []byte{}
	}
	return bytesBuffer.Bytes()
}
//3.生成哈希，目前不再使用此方法，而是使用pow
//序列化
func (block *Block) Serialize() []byte {
	var buffer bytes.Buffer

	//- 使用gob进行序列化（编码）得到字节流
	//1. 定义一个编码器
	//2. 使用编码器进行编码
	encoder := gob.NewEncoder(&buffer)
	err := encoder.Encode(&block)
	if err != nil {
		log.Panic("编码出错!")
	}

	//fmt.Printf("编码后的小明：%v\n", buffer.Bytes())

	return buffer.Bytes()
}

//反序列化
func Deserialize(data []byte) Block {

	decoder := gob.NewDecoder(bytes.NewReader(data))

	var block Block
	//2. 使用解码器进行解码
	err := decoder.Decode(&block)
	if err != nil {
		log.Panic("解码出错!")
	}
	return block
}

基于Bolt数据库实现可持续化的区块链

1.重构blockChain结构体，结构体变量为：1. bolt数据库对象db，2.存放最后一块区块的哈希tail。

type BlockChain struct {
	db *bolt.DB
	tail []byte
}

2.在block.go中实现能将结构体block系列化为字节的Serialize方法，和将字节反序列化为结构体的Deserialize方法。

//序列化
func (block *Block) Serialize() []byte {
	var buffer bytes.Buffer

	//- 使用gob进行序列化（编码）得到字节流
	//1. 定义一个编码器
	//2. 使用编码器进行编码
	encoder := gob.NewEncoder(&buffer)
	err := encoder.Encode(&block)
	if err != nil {
		log.Panic("编码出错!")
	}

	//fmt.Printf("编码后的小明：%v\n", buffer.Bytes())

	return buffer.Bytes()
}

//反序列化
func Deserialize(data []byte) Block {

	decoder := gob.NewDecoder(bytes.NewReader(data))

	var block Block
	//2. 使用解码器进行解码
	err := decoder.Decode(&block)
	if err != nil {
		log.Panic("解码出错!")
	}
	return block
}

2.重构newBlockChain方法，将创世块的哈希和序列化后的结构体，存放到boltDB中。并返回db对象，和最后一块区块哈希。

const blockChainDb = "blockChain.db"
const blockBucket = "blockBucket"

//5. 定义一个区块链
func NewBlockChain() *BlockChain {
	//return &BlockChain{
	//	blocks: []*Block{genesisBlock},
	//}

	//最后一个区块的哈希， 从数据库中读出来的
	var lastHash []byte

	//1. 打开数据库
	db, err := bolt.Open(blockChainDb, 0600, nil)
	//defer db.Close()

	if err != nil {
		log.Panic("打开数据库失败！")
	}

	//将要操作数据库（改写）
	_=db.Update(func(tx *bolt.Tx) error {
		//2. 找到抽屉bucket(如果没有，就创建）
		bucket := tx.Bucket([]byte(blockBucket))
		if bucket == nil {
			//没有抽屉，我们需要创建
			bucket, err = tx.CreateBucket([]byte(blockBucket))
			if err != nil {
				log.Panic("创建bucket(b1)失败")
			}

			//创建一个创世块，并作为第一个区块添加到区块链中
			genesisBlock := GenesisBlock()

			//3. 写数据
			//hash作为key， block的字节流作为value，尚未实现
			_=bucket.Put(genesisBlock.Hash, genesisBlock.Serialize())
			_=bucket.Put([]byte("LastHashKey"), genesisBlock.Hash)
			lastHash = genesisBlock.Hash

			////这是为了读数据测试，马上删掉,套路!
			//blockBytes := bucket.Get(genesisBlock.Hash)
			//block := Deserialize(blockBytes)
			//fmt.Printf("block info : %s\n", block)

		} else {
			lastHash = bucket.Get([]byte("LastHashKey"))
		}

		return nil
	})

	return &BlockChain{db, lastHash}
}

3.重构blockChain结构体的AddBlock方法，获取lastHash，然后在函数中执行Newblock方法，并将该block的hash和信息序列化到boltDB中。

func (bc *BlockChain) AddBlock(data string) {
	//如何获取前区块的哈希呢？？
	db := bc.db //区块链数据库
	lastHash := bc.tail //最后一个区块的哈希

	_=db.Update(func(tx *bolt.Tx) error {

		//完成数据添加
		bucket := tx.Bucket([]byte(blockBucket))
		if bucket == nil {
			log.Panic("bucket 不应该为空，请检查!")
		}


		//a. 创建新的区块
		block := NewBlock(data, lastHash)

		//b. 添加到区块链db中
		//hash作为key， block的字节流作为value，尚未实现
		_=bucket.Put(block.Hash, block.Serialize())
		_=bucket.Put([]byte("LastHashKey"), block.Hash)

		//c. 更新一下内存中的区块链，指的是把最后的小尾巴tail更新一下
		bc.tail = block.Hash

		return nil
	})
}
4.定义迭代器，方便后续遍历。

package main

import (
	"github.com/boltdb/bolt"
	"log"
)

type BlockChainIterator struct {
	db *bolt.DB
	//游标，用于不断索引
	currentHashPointer []byte
}

//func NewIterator(bc *BlockChain)  {
//
//}

func (bc *BlockChain) NewIterator() *BlockChainIterator {
	return &BlockChainIterator{
		bc.db,
		//最初指向区块链的最后一个区块，随着Next的调用，不断变化
		bc.tail,
	}
}

//迭代器是属于区块链的
//Next方式是属于迭代器的
//1. 返回当前的区块
//2. 指针前移
func (it *BlockChainIterator) Next() *Block {
	var block Block
	_=it.db.View(func(tx *bolt.Tx) error {
		bucket := tx.Bucket([]byte(blockBucket))
		if bucket == nil {
			log.Panic("迭代器遍历时bucket不应该为空，请检查!")
		}

		blockTmp := bucket.Get(it.currentHashPointer)
		//解码动作
		block = Deserialize(blockTmp)
		//游标哈希左移
		it.currentHashPointer = block.PrevHash

		return nil
	})

	return &block
}
func (it *BlockChainIterator)Restore(){
	//用于将迭代器游标移回初始值位置
	_=it.db.View(func(tx *bolt.Tx) error {
		bucket := tx.Bucket([]byte(blockBucket))
		if bucket == nil {
			log.Panic("迭代器遍历时bucket不应该为空，请检查!")
		}

		blockTmp := bucket.Get([]byte("LastHashKey"))
		//游标哈希左移
		it.currentHashPointer = blockTmp

		return nil
	})
}

5.新建cli.go文件用于接收外部参数，执行命令。

package main

import (
	"flag"
	"fmt"
	"os"
)

const Usage  = `
	AddBlock --data  	"add block to blockChain" 	example:./block AddBlock {DATA}
	PrintBlockChain     "print all blockChain data"
`
const AddBlockString  ="AddBlock"
const PrintBlockString  = "PrintBlockChain"
type Cli struct {
	Bc *BlockChain
}
func PrintUsage (){
	println(Usage)
}
func (cli *Cli)CheckInputLenth(){
	if len(os.Args)<2{
		fmt.Println("Invalid ARGS")
		PrintUsage()
		os.Exit(1)
	}
}
func (this *Cli)Run(){
	this.CheckInputLenth()
	AddBlocker:=flag.NewFlagSet(AddBlockString,flag.ExitOnError)
	PrintBlockChainer:=flag.NewFlagSet(PrintBlockString,flag.ExitOnError)
	AddBlockerParam:=AddBlocker.String("data","","AddBlock {data}")
	switch os.Args[1] {
	case AddBlockString:
		//AddBlock
		err:=AddBlocker.Parse(os.Args[2:])
		if err!=nil{fmt.Println(err)}
		if AddBlocker.Parsed(){
			if *AddBlockerParam==""{PrintUsage()}else {
				this.AddBlock(*AddBlockerParam)
			}
		}
	case PrintBlockString:
		//PrintBlockChain
		err:=PrintBlockChainer.Parse(os.Args[2:])
		if err!=nil{fmt.Println(err)}
		if PrintBlockChainer.Parsed(){
			this.PrintBlockChain()
		}
	default:
		fmt.Println("Invalid input ")
		PrintUsage()
	}
}

6.新建command.go文件，用于实现cli的方法。

package main

import "fmt"

func (this *Cli)AddBlock(data string){
	this.Bc.AddBlock(data)
}
func (this *Cli)PrintBlockChain(){
	//TODO
	Iterator:=this.Bc.NewIterator()
	HeightblockChain:=0
	for{
		HeightblockChain++
		block:=Iterator.Next()
		if len(block.PrevHash)==0{
			Iterator.Restore()
			break
		}

	}

	for{
		block:=Iterator.Next()
		fmt.Printf("=======当前区块高度:%d======\n",HeightblockChain)
		fmt.Printf("当前哈希：%x\n",block.Hash)
		fmt.Printf("上一级哈希：%x\n",block.PrevHash)
		fmt.Printf("交易信息：%s\n",block.Data)
		HeightblockChain--
		if len(block.PrevHash)==0{
			break
		}
	}
}

7.修改main函数，

package main

/*
区块的创建分为7步
1.定义结构体
2.创建区块
3.生成哈希
4.定义区块链结构体
5.生成区块链并添加创世
6.生成创世块块
7.添加其他区块
*/

func main(){
	bc:=NewBlockChain()
	cli:=Cli{bc}
	cli.Run()
}



至此为止，基于bolt的区块链开发就结束了。后续我们将深入探讨区块链知识。
