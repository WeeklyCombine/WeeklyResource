## BCH 交易源码分析

    class COutPoint {

        TxId txid;//这里bch 与 btc 历史源码不同，将txid的 hash放在了外部
        uint32_t n;//来自上一个交易的第n个out

        template <typename Stream, typename Operation>
        inline void SerializationOp(Stream &s, Operation ser_action) {
            READWRITE(txid);
            READWRITE(n);
        }
    };


TxInput

    /*
        一笔交易中的单个输入定义  
        pre这个输入包含输入对应的前一笔交易的输出匹配前一笔交易输出公钥的签名
    */
    class CTxIn {
        COutPoint prevout;//前一笔交易
        CScript scriptSig;//由spender客户端中的钱包软件来生成，序列化的脚本
        uint32_t nSequence;//引入原因是出于限制 nlockTime 的使用

        //Ques: nSequence 的用途需要进一步了解 
        https://www.8btc.com/article/118717
        /*若交易中每笔输入的nSequence 都是 0xffffffff，那么nlockTime将无效*/
        static const uint32_t SEQUENCE_FINAL = 0xffffffff;
        /*这里的规则与Bitcoin的文档中描述不一致，待查看BIP 68*/
        /* Below flags apply in the context of BIP 68*/
        /**
        * If this flag set, CTxIn::nSequence is NOT interpreted as a relative
        * lock-time.
        */
        static const uint32_t SEQUENCE_LOCKTIME_DISABLE_FLAG = (1 << 31);

        /**
        * If CTxIn::nSequence encodes a relative lock-time and this flag is set,
        * the relative lock-time has units of 512 seconds, otherwise it specifies
        * blocks with a granularity of 1.
        */
        static const uint32_t SEQUENCE_LOCKTIME_TYPE_FLAG = (1 << 22);

        /**
        * If CTxIn::nSequence encodes a relative lock-time, this mask is applied to
        * extract that lock-time from the sequence field.
        */
        static const uint32_t SEQUENCE_LOCKTIME_MASK = 0x0000ffff;

        /**
        * In order to use the same number of bits to encode roughly the same
        * wall-clock duration, and because blocks are naturally limited to occur
        * every 600s on average, the minimum granularity for time-based relative
        * lock-time is fixed at 512 seconds. Converting from CTxIn::nSequence to
        * seconds is performed by multiplying by 512 = 2^9, or equivalently
        * shifting up by 9 bits.
        */
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
