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


TxOutput

    class CTxOut {
        Amount nValue;
        CScript scriptPubKey;
    /*
        计算手续费 聪/千字节
        一笔非隔离见证下的标准支出大小为 34 字节，输入的大小为 148 字节
        这里说明了 隔离见证条件下的交易字节数明显少于非隔离见证条件下一条交易的总长度
    */
    Amount GetDustThreshold(const CFeeRate &minRelayTxFee) const {
        /**
         * "Dust" is defined in terms of CTransaction::minRelayTxFee, which has
         * units satoshis-per-kilobyte. If you'd pay more than 1/3 in fees to
         * spend something, then we consider it dust. A typical spendable
         * non-segwit txout is 34 bytes big, and will need a CTxIn of at least
         * 148 bytes to spend: so dust is a spendable txout less than
         * 546*minRelayTxFee/1000 (in satoshis). A typical spendable segwit
         * txout is 31 bytes big, and will need a CTxIn of at least 67 bytes to
         * spend: so dust is a spendable txout less than 294*minRelayTxFee/1000
         * (in satoshis).
         */
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


Transaction

交易的生成-- createTransaction

 * 计算全部支出的金额
 * 从钱包中获取可用的UTXO
 * 计算手续费 
   * 计算每笔支出减去分摊的手续费后，是否是dust交易
   * 如果交易额足够大，添加到支出队列中
 * 从UTXO池中选择用于支付的支出
 * 计算credit的币龄，并用于计算优先级
