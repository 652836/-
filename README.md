# 用户流失预警体系

## 1. 业务问题定义  
在电商场景下，用户流失通常定义为用户长时间未登录或活跃。在此处规定“连续7天未登录即为流失”，并考虑动态定义：对于**高频用户**（例如每日活跃）缩短这一阈值，对**低频用户**（每周或每月购买）适当延长。在运营目标上，以降低整体流失率10%为基本目标，因此重点对高风险用户进行预警和干预。

## 2. 数据理解与处理  
- **数据集来源**：使用“Kartikey Bartwal 电商推荐数据集”（Kaggle）作为样例。该数据集包含用户购买/浏览日志、商品分类等信息，可用于构造用户行为特征。  
- **特征设计**：结合业务，需要构造多维特征，包括：  
  - **人口属性**：用户性别、年龄段、所在城市。  
  - **行为特征**：最近登录距离（天）、历史总购买次数、购买频次（单位时间内）、总支出金额、兴趣类别分布（购买或浏览过的商品分类）、页面浏览数、停留时间。  
  - **RFM特征**：最近一次购买距离、购买频次、总消费额，用于用户价值评估。  
- **数据切分**：按帕累托原则，对数据集进行划分，80%作为训练集，20%作为测试集，避免信息泄露，在训练集中，删除用于构造流失标签的字段'Last_Login_Days_Ago'。
- **预处理步骤**：清洗重复或缺失记录，统一时间格式，去除异常值（如极端高消费）。对连续特征进行归一化或分箱处理，对类别特征（如兴趣标签）进行独热编码。  
- **数据局限性**：这个数据集类似一种静态快照（Static Snapshot），只是保留了某一个时刻的具体数据，不足以支撑需要的时序分析和窗口滑动。仅有一个字段'Last_Login_Days_Ago'可以作为时间段构造，与目标变量流失的构造字段重合，可能会造成数据泄露，因为本分析采用常见分类模型（RandomForest、XGBoost、神经网络、LightGBM等）来拟合最为合适。此外，当前数据样本规模较小，且用户“流失/未流失”类别比例不平衡，数据量过少且类别不平衡会导致模型效果不佳​。为缓解这一问题，我对训练数据采用了过采样方法（如SMOTE）来生成少数类样本并平衡类别分布​imbalanced-learn.org。这样可使模型在训练时获得更多流失样本，从而减轻类别偏差的影响。（详细对比可见notebook）


## 3. 建模方法与评估   

- **机器学习模型**：同时构建多种分类模型比较效果：  
  - **决策树/随机森林**：易解释，对特征重要性敏感，可处理高维混合特征，但易过拟合。适合初期基准模型。  
  - **XGBoost**：梯度提升树，强大的泛化能力和非线性拟合能力，常用于二分类预测，可通过特征重要性解释结果。  
  - **简单神经网络**：多层感知机等深度学习模型，能捕捉复杂非线性关系，但需大数据量且对超参较敏感。
  - **LightGBM**:高效的梯度提升框架，在大规模数据集上的训练速度比其他梯度提升框架快得多，且具有良好的准确性和泛化能力，但在小样本数据上可能会出现过拟合的情况。
- **评价指标**：选用多种指标评估模型效果：   
  - **精确率（Precision）**：预测为流失用户中实际流失的比例，关注干预资源利用效率 ([Building a Churn Prediction Model (Theoratical Guide) | by Amit Yadav | Biased-Algorithms | Medium](https://medium.com/biased-algorithms/building-a-churn-prediction-model-e8558add21a4#:~:text=,predicting))。  
  - **召回率（Recall）**：实际流失用户中被正确预测的比例，关注捕捉流失客户的能力 ([Building a Churn Prediction Model (Theoratical Guide) | by Amit Yadav | Biased-Algorithms | Medium](https://medium.com/biased-algorithms/building-a-churn-prediction-model-e8558add21a4#:~:text=,predicting))。  
  - **Recall@K**：对预测风险排序，关注前K高风险用户中实际流失用户的比例，常用于评估重点挽回效果 ([Precision and recall at K in ranking and recommendations](https://www.evidentlyai.com/ranking-metrics/precision-recall-at-k#:~:text=,within%20the%20top%20K%20positions)) ([Building a Churn Prediction Model (Theoratical Guide) | by Amit Yadav | Biased-Algorithms | Medium](https://medium.com/biased-algorithms/building-a-churn-prediction-model-e8558add21a4#:~:text=,predicting))。  
  - **F1_score**: 精确率和召回率的调和平均，关注模型整体性能。 

## 4. RFM分析与流失风险  
基于RFM（最近一次购买Recency、购买频次Frequency、消费金额Monetary）对用户进行分层，可挖掘不同价值群体的流失规律 ([IJIKM - A Novel Telecom Customer Churn Analysis System Based on RFM Model and Feature Importance Ranking](https://www.informingscience.org/Publications/5192#:~:text=Methodology%20The%20telecom%20customer%20churn,7%2C043%20instances%20and%2021%20features))。常见方法是将用户按R、F、M打分（如1–5分），或结合K-Means聚类进行自动分群。RFM分析可作为一种无监督分群手段，帮助构造额外特征或确定高价值用户群。如Xu等人在电信流失分析中，采用RFM模型与K-Means对用户分层后，再基于RFM特征构造预测变量，结合XGBoost等模型显著提升预测准确度 ([IJIKM - A Novel Telecom Customer Churn Analysis System Based on RFM Model and Feature Importance Ranking](https://www.informingscience.org/Publications/5192#:~:text=Methodology%20The%20telecom%20customer%20churn,7%2C043%20instances%20and%2021%20features))。  

一般而言，R、F、M较高的用户价值高、流失风险相对较低；反之，则需要更多挽回关注。因此通过RFM分层后，基于分层结果定义流失情况：对于高价值客户若7天未来访则视为流失，而低价值客户宽松到14天。明确问题后，在数据中标记**流失事件**（是否流失）作为预测目标。这里基于下表为基于RFM分层的用户特征和建议策略：  

| 用户分层 (RFM特征)        | 特征描述               | 流失风险 | 建议挽回策略       |
|---------------------------|-----------------------|----------|------------------|
| **高价值用户** (R近，F高，M高)  | 最近活跃、购买频繁、消费大   | 较低     | VIP专属优惠、个性化推荐、关怀邮件 |
| **中等价值用户** (R中，F中，M中) | 近期有消费、购买正常、消费一般 | 中      | 个性化优惠券、活动提醒邮件     |
| **低价值用户** (R远，F低，M低)   | 长期未消费、购买稀少、消费低   | 较高     | 低门槛折扣券、免费试用或再营销通知 |

（注：表中“R近”表示最近一次购买时间距离较短，“F高”表示购买次数多，“M高”表示消费金额大。）  

## 5. 挽回策略设计  
针对不同RFM层次的**高流失风险用户**设计差异化挽回方案，目标总体降低10%流失率。主要策略包括：  
- **个性化优惠与奖励**：对高价值且风险上升的用户，可提供VIP折扣、专属优惠券或增值服务（如延长会员权益）。研究表明，针对性优惠券活动能有效刺激低迷客户回归 ([Increase Retention and Prevent Churn with Personalized Coupons | Twilio Segment](https://segment.com/recipes/voucherify-increase-retention-prevent-churn-coupons/#:~:text=effective%20personalized%20coupon%20campaign%20to,customer%20churn%20and%20increase%20retention))。  
- **定向推荐和内容推送**：通过分析用户兴趣标签，推送个性化商品或内容，保持用户粘性。例如对中价值用户推荐符合其购买历史的新产品，增强购买动力。个性化互动可以显著降低流失率，使用户感受到价值和关怀 ([Maximizing Customer Retention with Personalized Content Strategies](https://www.storyly.io/post/using-personalized-content-for-customer-retention-strategies-and-benefits#:~:text=match%20at%20L191%20personalized%20interactions%2C,stable%20and%20loyal%20customer%20base))。  
- **提醒与关怀邮件**：对高风险用户发送周期性关怀短信/邮件，如活动预告、产品使用指南或感谢信，提醒他们品牌存在。如提醒长期未登录用户查看新上线功能或提供提醒服务。  
- **忠诚度计划**：建立积分或会员体系，对活跃度高的用户给予奖励，培养用户忠诚。对于高价值用户，可邀请加入VIP俱乐部，提供专属客服或体验优惠。  
- **广告再营销**：对于长时间未活跃的用户，使用社交媒体或电商站内广告召回，展示热门商品或限时优惠信息，吸引其重新访问。  

上述策略需要结合业务渠道（邮件、推送、短信等）实施，并进行A/B测试（需要对应数据）评估效果。总体目标是通过**差异化策略**提高挽回转化率，结合预测模型动态调整策略。  


## 6.结果分析
在当前数据和特征条件下，各模型在测试集上的**整体准确率约为60%**左右。值得注意的是，对“未流失”这一多数类的预测准确率也相对较低。主要原因是：数据量太小且特征不足，难以捕捉复杂的流失模式​。其次，不平衡的数据加剧了学习偏差，即使使用了SMOTE，在随机森林和神经网络上确实提升了未流失的用户的预测精度，但是会相对减少模型对于流失用户特征的关注，总体性能下降。最后，不同模型表现参差，XGBoost和LightGBM本身有平衡类别的机制，所以上采样对于他们来说并不必要。

总的来说，本次模型构建在数据规模和特征丰富度方面存在局限，导致当前预测效果仅能达到参考水平。对于实际业务而言，还需积累更多用户数据和多样化特征，并在此基础上持续优化模型。并且结合更多业务指标辅助判断，同时加大对关键用户的关注。通过上述改进，企业可在未来实现用户流失风险的更准确预警和干预，真正发挥预测模型的价值。


