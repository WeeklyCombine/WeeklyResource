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

![](https://github.com/WeeklyCombine/WeeklyResource/blob/master/doc/resources/CreateTransaction_1.png "创建交易")

instruction：
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


    //创建交易
    bool CWallet::CreateTransaction(const std::vector<CRecipient> &vecSend,//接收转账的对象队列
                                    CWalletTx &wtxNew, 
                                    CReserveKey &reservekey,//私钥池中的私钥
                                    Amount &nFeeRet, //Amout 一个计数结构
                                    int &nChangePosInOut,
                                    std::string &strFailReason,
                                    const CCoinControl *coinControl, //to do:暂时不明确
                                    bool sign) {//是否签名
        Amount nValue(0);//用于保存即将转出的金额
        int nChangePosRequest = nChangePosInOut;
        unsigned int nSubtractFeeFromAmount = 0;
        //计算总共支出的金额
        for (const auto &recipient : vecSend) {
            if (nValue < Amount(0) || recipient.nAmount < Amount(0)) {//UTXO可用额度异常
                strFailReason = _("Transaction amounts must not be negative");
                return false;
            }

            nValue += recipient.nAmount;//计算在队列中的支出总额度

            if (recipient.fSubtractFeeFromAmount) {//to do:判断该笔支出是否已经减去手续费
                nSubtractFeeFromAmount++;
            }
        }

        //？why not check it first
        if (vecSend.empty()) {
            strFailReason = _("Transaction must have at least one recipient");
            return false;
        }

        wtxNew.fTimeReceivedIsTxTime = true;
        wtxNew.BindWallet(this);
        CMutableTransaction txNew;

        //https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2013-February/002185.html
        //以上链接能够帮助理解一下 fee sniping 的问题
        // Discourage fee sniping. 
        if (GetRandInt(10) == 0) {
            txNew.nLockTime = std::max(0, (int)txNew.nLockTime - GetRandInt(100));
        }

        {
            std::set<std::pair<const CWalletTx *, uint32_t>> setCoins;
            LOCK2(cs_main, cs_wallet);

            std::vector<COutput> vAvailableCoins;
            AvailableCoins(vAvailableCoins, true, coinControl);

            nFeeRet = Amount(0);//手续费总额
            // Start with no fee and loop until there is enough fee.
            while (true) {
                //用于重新构建交易
                nChangePosInOut = nChangePosRequest;
                txNew.vin.clear();
                txNew.vout.clear();
                wtxNew.fFromMe = true;
                bool fFirst = true;
                //nValueToSelect 记录需要匹配的输入额度
                Amount nValueToSelect = nValue;
                if (nSubtractFeeFromAmount == 0) {
                    nValueToSelect += nFeeRet;
                }

                double dPriority = 0;
                // vouts to the payees 计算每笔转账手续费
                for (const auto &recipient : vecSend) {
                    CTxOut txout(recipient.nAmount, recipient.scriptPubKey);

                    if (recipient.fSubtractFeeFromAmount) {
                        // Subtract fee equally from each selected recipient.
                        txout.nValue -= nFeeRet / int(nSubtractFeeFromAmount);

                        // First receiver pays the remainder not divisible by output
                        // count.
                        if (fFirst) {
                            fFirst = false;
                            txout.nValue -= nFeeRet % int(nSubtractFeeFromAmount);
                        }
                    }

                    if (txout.IsDust(dustRelayFee)) {//判断是否是dust交易
                        if (recipient.fSubtractFeeFromAmount &&
                            nFeeRet > Amount(0)) {
                            if (txout.nValue < Amount(0)) {
                                strFailReason = _("The transaction amount is "
                                                "too small to pay the fee");
                            } else {
                                strFailReason =
                                    _("The transaction amount is too small to "
                                    "send after the fee has been deducted");
                            }
                        } else {
                            strFailReason = _("Transaction amount too small");
                        }

                        return false;
                    }

                    txNew.vout.push_back(txout);
                }

                // 选择可以用于支付的入账 
                Amount nValueIn(0);
                setCoins.clear();
                if (!SelectCoins(vAvailableCoins,//UTXO池 
                                 nValueToSelect, //需要匹配的输出总额
                                 setCoins,//返回选择结果
                                 nValueIn, //返回输入总额
                                 coinControl)) {//to do: 选择入账待详细解析
                    strFailReason = _("Insufficient funds");
                    return false;
                }
                //计算优先级
                for (const auto &pcoin : setCoins) {
                    Amount nCredit = pcoin.first->tx->vout[pcoin.second].nValue;
                    int age = pcoin.first->GetDepthInMainChain();
                    assert(age >= 0);
                    if (age != 0) age += 1;
                    dPriority += (double)nCredit.GetSatoshis() * age;
                }

                const Amount nChange = nValueIn - nValueToSelect;//找零
                if (nChange > Amount(0)) {
                    //选择新地址还是复用原来的地址各有利弊，但原则上为了安全尽量使用新的地址
                    // Coin control: send change to custom address.
                    if (coinControl &&
                        !boost::get<CNoDestination>(&coinControl->destChange)) {
                        scriptChange =
                            GetScriptForDestination(coinControl->destChange);

                        // No coin control: send change to newly generated address.
                    } else {
                        // Reserve a new key pair from key pool.
                        CPubKey vchPubKey;
                        bool ret;
                        ret = reservekey.GetReservedKey(vchPubKey, true);
                        if (!ret) {
                            strFailReason = _("Keypool ran out, please call "
                                            "keypoolrefill first");
                            return false;
                        }

                        scriptChange = GetScriptForDestination(vchPubKey.GetID());
                    }

                    CTxOut newTxOut(nChange, scriptChange);

                    //调整找零至满足手续费并不会过高
                    if (nSubtractFeeFromAmount > 0 &&
                        newTxOut.IsDust(dustRelayFee)) {
                        Amount nDust = newTxOut.GetDustThreshold(dustRelayFee) -
                                    newTxOut.nValue;
                        // Raise change until no more dust.
                        newTxOut.nValue += nDust;
                        // Subtract from first recipient.
                        for (unsigned int i = 0; i < vecSend.size(); i++) {
                            if (vecSend[i].fSubtractFeeFromAmount) {
                                txNew.vout[i].nValue -= nDust;
                                if (txNew.vout[i].IsDust(dustRelayFee)) {
                                    strFailReason =
                                        _("The transaction amount is too small "
                                        "to send after the fee has been "
                                        "deducted");
                                    return false;
                                }

                                break;
                            }
                        }
                    }

                    //判断找零是否足够大
                    if (newTxOut.IsDust(dustRelayFee)) {
                        nChangePosInOut = -1;
                        nFeeRet += nChange;
                        reservekey.ReturnKey();
                    } else {
                        if (nChangePosInOut == -1) {
                            // Insert change txn at random position:
                            nChangePosInOut = GetRandInt(txNew.vout.size() + 1);
                        } else if ((unsigned int)nChangePosInOut >
                                txNew.vout.size()) {
                            strFailReason = _("Change index out of range");
                            return false;
                        }

                        std::vector<CTxOut>::iterator position =
                            txNew.vout.begin() + nChangePosInOut;
                        txNew.vout.insert(position, newTxOut);
                    }
                } else {
                    reservekey.ReturnKey();
                }

                //手续费计算完毕后，填充交易
                for (const auto &coin : setCoins) {
                    txNew.vin.push_back(
                        CTxIn(coin.first->GetId(), coin.second, CScript(),
                            std::numeric_limits<uint32_t>::max() - 1));
                }

                //填入虚拟签名
                // Fill in dummy signatures for fee calculation.
                if (!DummySignTx(txNew, setCoins)) {
                    strFailReason = _("Signing transaction failed");
                    return false;
                }

                CTransaction txNewConst(txNew);
                unsigned int nBytes = txNewConst.GetTotalSize();
                dPriority = txNewConst.ComputePriority(dPriority, nBytes);

                //移除dummy签名
                for (auto &vin : txNew.vin) {
                    vin.scriptSig = CScript();
                }

                int currentConfirmationTarget = nTxConfirmTarget;
                if (coinControl && coinControl->nConfirmTarget > 0) {
                    currentConfirmationTarget = coinControl->nConfirmTarget;
                }

                //估算手续费
                Amount nFeeNeeded =
                    GetMinimumFee(nBytes, currentConfirmationTarget, mempool);//最小手续费 
                if (coinControl && nFeeNeeded > Amount(0) &&
                    coinControl->nMinimumTotalFee > nFeeNeeded) {
                    nFeeNeeded = coinControl->nMinimumTotalFee;
                }

                if (coinControl && coinControl->fOverrideFeeRate) {
                    nFeeNeeded = coinControl->nFeeRate.GetFee(nBytes);
                }

                //这里的 relay fee 暂时不明是具体产生的原因
                // If we made it here and we aren't even able to meet the relay fee
                // on the next pass, give up because we must be at the maximum
                // allowed fee.
                Amount minFee = GetConfig().GetMinFeePerKB().GetFee(nBytes);//计算标准最小手续费
                if (nFeeNeeded < minFee) {//如果无法满足标准最小手续费，需要重组交易
                    strFailReason = _("Transaction too large for fee policy");
                    return false;
                }

                //根据估算的手续费 和 计算出的手续费，增加找零，减少计算出的手续费额度
                if (nFeeRet >= nFeeNeeded) {
                    if (nFeeRet > nFeeNeeded && nChangePosInOut != -1 &&
                        nSubtractFeeFromAmount == 0) {
                        Amount extraFeePaid = nFeeRet - nFeeNeeded;
                        std::vector<CTxOut>::iterator change_position =
                            txNew.vout.begin() + nChangePosInOut;
                        change_position->nValue += extraFeePaid;
                        nFeeRet -= extraFeePaid;
                    }

                    // Done, enough fee included.
                    break;
                }

                //如果手续费不足，减少找零以满足手续费，前提是减少后找零不是dust，所以有大小限制
                if (nChangePosInOut != -1 && nSubtractFeeFromAmount == 0) {
                    Amount additionalFeeNeeded = nFeeNeeded - nFeeRet;
                    std::vector<CTxOut>::iterator change_position =
                        txNew.vout.begin() + nChangePosInOut;
                    // Only reduce change if remaining amount is still a large
                    // enough output.
                    if (change_position->nValue >=
                        MIN_FINAL_CHANGE + additionalFeeNeeded) {
                        change_position->nValue -= additionalFeeNeeded;
                        nFeeRet += additionalFeeNeeded;
                        // Done, able to increase fee from change.
                        break;
                    }
                }

                //走到这里说明，手续费不够，需要重新构建交易
                // Include more fee and try again.
                nFeeRet = nFeeNeeded;
                continue;
            }

            //是否已签名
            if (sign) {
                SigHashType sigHashType = SigHashType().withForkId();

                CTransaction txNewConst(txNew);
                int nIn = 0;
                for (const auto &coin : setCoins) {
                    const CScript &scriptPubKey =
                        coin.first->tx->vout[coin.second].scriptPubKey;
                    SignatureData sigdata;

                    if (!ProduceSignature(
                            TransactionSignatureCreator(
                                this, &txNewConst, nIn,
                                coin.first->tx->vout[coin.second].nValue,
                                sigHashType),
                            scriptPubKey, sigdata)) {
                        strFailReason = _("Signing transaction failed");
                        return false;
                    } else {
                        UpdateTransaction(txNew, nIn, sigdata);
                    }

                    nIn++;
                }
            }

            // Embed the constructed transaction data in wtxNew.
            wtxNew.SetTx(MakeTransactionRef(std::move(txNew)));
            //如果现在打包的交易的大小超过限制，需要重新打包交易
            // Limit size.
            if (CTransaction(wtxNew).GetTotalSize() >= MAX_STANDARD_TX_SIZE) {
                strFailReason = _("Transaction too large");
                return false;
            }
        }

        if (gArgs.GetBoolArg("-walletrejectlongchains",
                            DEFAULT_WALLET_REJECT_LONG_CHAINS)) {
            // Lastly, ensure this tx will pass the mempool's chain limits.
            LockPoints lp;
            CTxMemPoolEntry entry(wtxNew.tx, Amount(0), 0, 0, 0, Amount(0), false,
                                0, lp);
            CTxMemPool::setEntries setAncestors;
            size_t nLimitAncestors =
                gArgs.GetArg("-limitancestorcount", DEFAULT_ANCESTOR_LIMIT);
            size_t nLimitAncestorSize =
                gArgs.GetArg("-limitancestorsize", DEFAULT_ANCESTOR_SIZE_LIMIT) *
                1000;
            size_t nLimitDescendants =
                gArgs.GetArg("-limitdescendantcount", DEFAULT_DESCENDANT_LIMIT);
            size_t nLimitDescendantSize =
                gArgs.GetArg("-limitdescendantsize",
                            DEFAULT_DESCENDANT_SIZE_LIMIT) *
                1000;
            std::string errString;
            if (!mempool.CalculateMemPoolAncestors(
                    entry, setAncestors, nLimitAncestors, nLimitAncestorSize,
                    nLimitDescendants, nLimitDescendantSize, errString)) {
                strFailReason = _("Transaction has too long of a mempool chain");
                return false;
            }
        }
        return true;
    }






