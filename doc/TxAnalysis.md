## BCH 交易源码分析

Outpoint

    class COutPoint {

        TxId txid;//这里bch 与 btc 历史源码不同，将txid的 hash放在了外部
        uint32_t n;//来自上一个交易的第n个out

        template <typename Stream, typename Operation>
        inline void SerializationOp(Stream &s, Operation ser_action) {
            READWRITE(txid);
            READWRITE(n);
        }
    };


TxOutput

    class CTxOut {
        Amount nValue;//这笔输出在输出队列中的index
        CScript scriptPubKey;//锁定脚本
        /*
            计算手续费 聪/千字节
            一笔非隔离见证下的标准支出大小为 34 字节，输入的大小为 148 字节
            这里说明了 隔离见证条件下的交易字节数明显少于非隔离见证条件下一条交易的总长度
        */
        Amount GetDustThreshold(const CFeeRate &minRelayTxFee) const {
            if (scriptPubKey.IsUnspendable()) return Amount(0);

            size_t nSize = GetSerializeSize(*this, SER_DISK, 0);

            // the 148 mentioned above
            nSize += (32 + 4 + 1 + 107 + 4);

            return 3 * minRelayTxFee.GetFee(nSize);
        }
    };


TxInput

    /*
        一笔交易中的单个输入定义  
        这个输入包含输入对应的前一笔交易的输出匹配前一笔交易输出公钥的签名
    */
    class CTxIn {
        COutPoint prevout;//前一笔交易
        CScript scriptSig;//由spender客户端中的钱包软件来生成，序列化的脚本
        uint32_t nSequence;//引入原因是出于限制 nlockTime 的使用

        //Ques: nSequence 的用途需要进一步了解 
        https://www.8btc.com/article/118717
        /*若交易中每笔输入的nSequence 都是 0xffffffff，那么nlockTime将无*/
        static const uint32_t SEQUENCE_FINAL = 0xffffffff;
        /*这里的规则与Bitcoin的文档中描述不一致，待查看BIP 68*/
        static const uint32_t SEQUENCE_LOCKTIME_DISABLE_FLAG = (1 << 31);
        static const uint32_t SEQUENCE_LOCKTIME_TYPE_FLAG = (1 << 22);
        static const uint32_t SEQUENCE_LOCKTIME_MASK = 0x0000ffff;
        static const int SEQUENCE_LOCKTIME_GRANULARITY = 9;

        //默认构造一笔输入的时候是没有时间锁定的
        CTxIn() { nSequence = SEQUENCE_FINAL; }
        /*
            param 
            prevoutIn      前一笔交易的输出 
            scriptSigIn    解锁脚本
            nSequenceIn     
        */
        explicit CTxIn(COutPoint prevoutIn, 
                       CScript scriptSigIn = CScript(),
                       uint32_t nSequenceIn = SEQUENCE_FINAL);

        CTxIn(TxId prevTxId, 
              uint32_t nOut, //前一笔输出在队列中的序号
              CScript scriptSigIn = CScript(),
              uint32_t nSequenceIn = SEQUENCE_FINAL);
    };


Transaction

    /**
    * The basic transaction that is broadcasted on the network and contained in
    * blocks. A transaction can contain multiple inputs and outputs.
    */
    // 区块中包含的交易的定义
    class CTransaction {
    public:
        // Default transaction version.
        static const int32_t CURRENT_VERSION = 2;
        static const int32_t MAX_STANDARD_VERSION = 2;
        /*交易的数据构成*/
        const int32_t nVersion;//涉及到分叉的时候如何来区分不同版本的客户端生成的交易
        const std::vector<CTxIn> vin;//交易发起人的输入队列
        const std::vector<CTxOut> vout;//交易发起人的输出队列
        const uint32_t nLockTime;//交易锁定时间

    public:
        //输入的优先级
        double ComputePriority(double dPriorityInputs,
                            unsigned int nTxSize = 0) const;

        //计算修改后的交易字节大小
        unsigned int CalculateModifiedSize(unsigned int nTxSize = 0) const;
        //获取交易的完整字节数
        unsigned int GetTotalSize() const;
        //是否是base交易
        bool IsCoinBase() const {
            return (vin.size() == 1 && vin[0].prevout.IsNull());
        }
    };

---
交易的生成-- createTransaction

创建交易过程中需要做的事情

![]()

说明：
 * 计算全部支出的金额
 * 从钱包中获取可用的UTXO
 * 计算手续费 
   * 计算每笔支出减去分摊的手续费后，是否是dust交易
   * 如果交易额足够大，添加到支出队列中
 * 从UTXO池中选择用于支付的支出
 * 计算credit的币龄，并用于计算优先级
 * 复用spender的签名脚本，并构建一笔找零
 * 计算添加找零之后的手续费，并可以对找零的数量进行调整以满足手续费的要求，如果无法满足则需要清空当前已构建的交易的输入输出，重新构建交易
 * 生成spender的sigScript配合公钥进行签名验证
 * 检查生成的交易的大小 

 ![](https://github.com/WeeklyCombine/WeeklyResource/blob/master/doc/resources/createTransaction.jpeg "创建交易流程")


源码





