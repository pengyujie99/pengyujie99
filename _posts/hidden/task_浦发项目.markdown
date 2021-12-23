### 浦发项目

需求：需要给浦发银行，上送我们对应的商户信息，只需要上送一个压缩文件（压缩文件包含 商户信息文件、商户昨日交易文件、商户资金结算文件），一个空信号文件（空文件最后.finish命名即可）。

上传到sftp之后，由浦发方去取数据。



#### 设置定时任务

这里创建三个定时任务，分别去生成对应的数据文件，在最后一个文件生成之后进行压缩上送。

~~~java
package com.lichu.common.task.t20;

import com.lichu.common.util.CommonUtil;
import com.lichu.common.util.DateUtils;
import com.lichu.common.util.PufaReceiptUtil;
import org.apache.log4j.Logger;
import org.springframework.context.annotation.Lazy;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;


/**
 * 浦发银行联合收单
 * 浦发银行上传商户和交易数据文件定时任务
 */
@Component
@Lazy(false)
public class PufaBankTask {
    private final static Logger log = Logger.getLogger(PufaBankTask.class);

    /**
     * 每天早上9:10点创建前一天的交易数据文件
     */

    /**
     * 生成商户信息同步文件
     */
//    @Scheduled(cron = "0 10 09 * * ?")
    @Scheduled(cron = "0 */2 * * * ?")
    public void PufaMerchantInfo(){

//        if(!CommonUtil.getTask_active()){
//            log.info("浦发银行生成交易明细信息同步文件定时任务开关处于关闭状态");
//            return;
//        }

        log.info("浦发银行生成商户信息同步文件,定时任务启动:"+ DateUtils.getCurrTime());
        PufaReceiptUtil.createInfoFile();
        log.info("浦发银行生成商户信息同步文件,定时任务结束:"+DateUtils.getCurrTime());
    }


    /**
     * 生成资金结算信息同步文件
     */
    @Scheduled(cron = "0 */2 * * * ?")
    public void PufaFundSettlement() {

//        if(!CommonUtil.getTask_active()){
//            log.info("浦发银行生成交易明细信息同步文件定时任务开关处于关闭状态");
//            return;
//        }

        log.info("浦发银行生成资金结算信息同步文件,定时任务启动:"+ DateUtils.getCurrTime());
        PufaReceiptUtil.createSettlementFile();
        log.info("浦发银行生成资金结算信息同步文件,定时任务结束:"+DateUtils.getCurrTime());
    }


    /**
     * 生成交易明细信息同步文件
     */
    @Scheduled(cron = "0 */2 * * * ?")
    public void PufaTransactionDetail(){

//        if(!CommonUtil.getTask_active()){
//            log.info("浦发银行生成交易明细信息同步文件定时任务开关处于关闭状态");
//            return;
//        }

        log.info("浦发银行生成交易明细信息同步文件,定时任务启动:"+ DateUtils.getCurrTime());
        PufaReceiptUtil.createDetailFile();
        log.info("浦发银行生成交易明细信息同步文件,定时任务结束:"+DateUtils.getCurrTime());


//        if(!CommonUtil.getTask_active()){
//            log.info("浦发银行上传商户和交易数据定时任务开关处于关闭状态");
//            return;
//        }

        log.info("浦发银行上传商户和交易数据,定时任务启动:"+ DateUtils.getCurrTime());
        PufaReceiptUtil.SynUpload();
        log.info("浦发银行上传商户和交易数据,定时任务结束:"+DateUtils.getCurrTime());
    }

}

~~~



#### 编写Bean类

~~~
FundSettlementSynBean  浦发资金结算同步数据
TransactionDetailSynBean  浦发交易明细同步数据
MerchantInfoSynBean  浦发商户信息同步数据
MerchantInfoSynOtherBean 商户表中需要展示的数据
~~~

这里由于只是笔记，bean类不进行展示。



#### 编写浦发文件生成工具类

在工具类中主要写四个大方法（生成三个文件的方法和最后压缩上送的方法）和其余转换传递参数的小方法

利用Stringbuffer进行拼接字符，然后`BufferedWriter bw = new BufferedWriter(new FileWriter(f))`利用`bw.bw.write(content.toString());`和`bw.flush();`将文件写入，最后关闭。

这样生成三个文件之后，利用压缩工具类将文件压缩到一个压缩文件进行上送。

文件生成和文件压缩的时候注意判断路径或文件是否存在，路径不存在则重新mkdirs，文件不存在就重新createNewFile，如果文件存在则直接return避免重新生成文件耗费资源。

~~~java
package com.lichu.common.util;


import com.lichu.common.bean.FundSettlementSynBean;
import com.lichu.common.bean.MerchantInfoSynBean;
import com.lichu.common.bean.MerchantInfoSynOtherBean;
import com.lichu.common.bean.TransactionDetailSynBean;
import com.lichu.common.service.MerchantService;
import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.*;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;


/**
 * 浦发银行联合收单工具类
 */
public class PufaReceiptUtil {

    private static final Logger logger = LoggerFactory.getLogger(PufaReceiptUtil.class);

    private static MerchantService merchantService = (MerchantService)SpringContextUtil.getBean("merchantService");

    private static List<String> list_tags = Arrays.asList("1","2","3","4","5");//湖北浦发、南昌浦发、长沙浦发、成都浦发、郑州浦发


    /**
     * 同步上送
     */
    public static void SynUpload(){
        SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd");//设置日期格式
        Calendar calendar = Calendar.getInstance();  //得到日历
        calendar.add(Calendar.DAY_OF_MONTH, -1);     //设置为昨天
        String yesterday = df.format(calendar.getTime());//得到昨天的的时间
        //生成压缩文件
        fileToZip(yesterday);
        //生成一个空信号文件
        createFinishFile();
        //并上传sftp
        try {
            SftpUtil.bacthUploadSPDFile(yesterday);
        } catch (Exception e) {
            logger.error("上传浦发商户文件到sftp异常:",e);
        }


    }



    /**
     * 生成昨日的商户信息文件
     */
    public static void createInfoFile(){
        File f = null;//新建文件

        SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd");//设置日期格式
        Calendar calendar = Calendar.getInstance();  //得到日历
        calendar.add(Calendar.DAY_OF_MONTH, -1);     //设置为昨天
        String yesterday = df.format(calendar.getTime());//得到昨天的的时间
        String localPath = "d:\\SPD_BANK\\"+yesterday+"\\";
        File path = new File(localPath);
        if (!path.exists()) {
            path.mkdirs();
        }

        String fileName ="12019900"+"_MCHT_INFO_"+yesterday;//生成文件名 合作机构代码（四位）+总/分行号（四位）_TXN_INFO_日期
        String csvFilePath = "";
        csvFilePath = localPath + fileName;
        f = new File(csvFilePath);//新建文件
        long begintime = DateUtils.getNow();
        try {
            if (f.exists()){
                return;
            }
            f.createNewFile();
            List<MerchantInfoSynBean> detailSynBean = merchantService.findInfoSynBean(list_tags);
            if (detailSynBean==null || detailSynBean.size()<1){
                return;
            }
            BufferedWriter bw = new BufferedWriter(new FileWriter(f));
            for (MerchantInfoSynBean bean : detailSynBean) {
                StringBuffer content = new StringBuffer();
                content.append(bean.getMerchantno());//内部商户号
                content.append("&&");
                content.append(bean.getMerchantno_uni());//银联商户号
                content.append("&&");
                content.append(spdMerchantStatus(bean.getStatus()));//商户状态
                content.append("&&");
                content.append(bean.getMerchantcompany());//商户法定名称
                content.append("&&");
                content.append(bean.getMerchantname());//商户经营名称
                content.append("&&");
                content.append(bean.getLicense_no());//营业证明文件号码
                content.append("&&");
                content.append(bean.getCardnum());//结算账号
                content.append("&&");
                content.append(spdSettlementType(bean.getAccounttype()));//结算账号类型
                content.append("&&");
                content.append(bean.getCardname());//结算账号户名
                content.append("&&");
                content.append("");//开户行名称
                content.append("&&");
                content.append("");//开户行行号
                content.append("&&");
                content.append(bean.getMerchantperson());//商户联系人姓名
                content.append("&&");
                content.append(bean.getMerchantphone());//商户联系电话
                content.append("&&");
                content.append(bean.getProvinceAndCity());//商户所在地市
                content.append("&&");
                content.append(bean.getMerchantaddress());//商户地址
                content.append("&&");
                content.append("");//开户日期
                content.append("&&");
                content.append(spdSignLineNum(bean.getSignLineNum()));//签约分行号
                content.append("&&");
                content.append(bean.getMerchantwx3id());//商户MCC
                content.append("&&");
                content.append("");//推荐人
                content.append("&&");
                content.append("");//pos借记卡刷卡费率
                content.append("&&");
                content.append(bean.getPos_borrow_value());//pos借记卡刷卡封顶值
                content.append("&&");
                content.append("");//pos贷记卡刷卡费率
                content.append("&&");
                content.append("");//pos贷记卡刷卡封顶值
                content.append("&&");
                content.append("");//微信费率
                content.append("&&");
                content.append("");//微信特殊费率
                content.append("&&");
                content.append("");//支付宝费率
                content.append("&&");
                content.append("");//支付宝特殊费率
                content.append("&&");
                content.append("");//银联扫码基准金额
                content.append("&&");
                content.append("");//银联扫码低分段费率(借记卡)
                content.append("&&");
                content.append("");//银联扫码低分段封顶值（借记卡）
                content.append("&&");
                content.append("");//银联扫码低分段费率(贷记卡)
                content.append("&&");
                content.append("");//银联扫码低分段封顶值（贷记卡）
                content.append("&&");
                content.append("");//银联扫码高分段费率(借记卡)
                content.append("&&");
                content.append("");//银联扫码高分段封顶值（借记卡）
                content.append("&&");
                content.append("");//银联扫码高分段费率(贷记卡)
                content.append("&&");
                content.append("");//银联扫码高分段封顶值（贷记卡）
                content.append("&&");
                content.append("");//预留字段1
                content.append("&&");
                content.append("");//预留字段2
                content.append("&&");
                content.append("");//预留字段3
                content.append("&&");
                content.append("");//预留字段4

                content.append("\n");//换行
                bw.write(content.toString());
                bw.flush();
            }
            bw.close();
            LichuSystemPushUtil.taskNotify("下游", "生成浦发商户信息文件", "浦发商户信息文件生成成功，耗时：" + (DateUtils.getNow() - begintime) / 1000 + "s");
        } catch (Exception e) {
            logger.error("生成浦发商户信息文件异常:",e);
            LichuSystemPushUtil.taskNotify("下游", "生成浦发商户信息文件异常", "生成浦发商户信息文件异常，请尽快处理，文件名：" + fileName);
        }

    }


    /**
     * 生成昨日的交易文件
     */
    public static void createDetailFile(){

        String yesterday = null;//得到昨天的的时间
        File f = null;//新建文件

        SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd");//设置日期格式
        Calendar calendar = Calendar.getInstance();  //得到日历
        calendar.add(Calendar.DAY_OF_MONTH, -1);     //设置为昨天
        yesterday = df.format(calendar.getTime());
        String localPath = "d:\\SPD_BANK\\"+yesterday+"\\";
        File path = new File(localPath);
        if (!path.exists()) {
            path.mkdirs();
        }

        String fileName ="12019900"+"_TXN_INFO_"+yesterday;//生成文件名 合作机构代码（四位）+总/分行号（四位）_TXN_INFO_日期
        String csvFilePath = "";
        csvFilePath = localPath + fileName;
        f = new File(csvFilePath);
        long begintime = DateUtils.getNow();
        try {
            if (f.exists()) {
                return;
            }
            f.createNewFile();
            //查询出商户表中需要上送的信息
            List<MerchantInfoSynOtherBean> merchants = merchantService.findMerchantNo(list_tags);
            if (merchants==null || merchants.size()<1){
                return;
            }
            BufferedWriter bw = new BufferedWriter(new FileWriter(f));

            for (MerchantInfoSynOtherBean merchant : merchants) {
                int pageSize = 50000;//查询大小
                Long id = 0L;//流水主键id,用于查询过滤
                Long count = 0L;
                //满足循环条件
                Boolean flag = true;
                while(flag) {//一次最多查50000条数据写入文件,不到50000条交易数据就写入之后查询下一个商户，如果不止就继续查出来写入文件
                    List<TransactionDetailSynBean> detailSynBean = merchantService.findDetailSynBean(yesterday, merchant.getMerchantid(), id, pageSize);
                    if (detailSynBean == null || detailSynBean.size() < 1) {
                        flag = false;
                        continue;
                    }
                    count = Long.parseLong(String.valueOf(detailSynBean.size()));//本次查询条数

                    for (TransactionDetailSynBean bean : detailSynBean) {
                        StringBuffer content = new StringBuffer();
                        content.append(bean.getTrade_date());//交易日期
                        content.append("&&");
                        content.append(bean.getTrade_time());//交易时间
                        content.append("&&");
                        content.append(bean.getSettlement_date());//清算日期
                        content.append("&&");
                        content.append(bean.getMerchantno());//内部商户号
                        content.append("&&");
                        content.append(bean.getMerchantno_uni());//银联商户号
                        content.append("&&");
                        content.append(bean.getTerminal_id());//终端号
                        content.append("&&");
                        content.append(spdTradeType(bean.getPay_mode(), bean.getCard_type(), bean.getTrade_type()));//支付方式
                        content.append("&&");
                        content.append(bean.getTrade_cardnum());//交易卡号
                        content.append("&&");
                        content.append("");//授权码
                        content.append("&&");
                        content.append(spdTradeType(bean.getTrade_type()));//交易类型
                        content.append("&&");
                        content.append(bean.getTotal_fee());//交易金额
                        content.append("&&");
                        content.append(bean.getRefund_fee());//退款金额
                        content.append("&&");
                        content.append(bean.getBalance());//清算金额
                        content.append("&&");
                        content.append("");//应收商户手续费
                        content.append("&&");
                        content.append("");//手续费减免金额
                        content.append("&&");
                        content.append("");//预留金额1
                        content.append("&&");
                        content.append("");//预留金额2
                        content.append("&&");
                        content.append(bean.getOut_trade_no());//系统流水号
                        content.append("&&");
                        content.append(bean.getTerminal_trace_no());//终端流水号
                        content.append("&&");
                        content.append("");//检索参考号
                        content.append("&&");
                        content.append("");//商户订单号
                        content.append("&&");
                        content.append(bean.getChannel_trade_no());//第三方订单号
                        content.append("&&");
                        content.append(spdOut_Refund_no(spdTradeType(bean.getTrade_type()), bean.getOut_trade_no()));//原商户订单号
                        content.append("&&");
                        content.append(spdOut_Refund_no(spdTradeType(bean.getTrade_type()), bean.getOut_trade_no()));//原系统流水号
                        content.append("&&");
                        content.append(bean.getRemarks());//交易备注
                        content.append("&&");
                        content.append(spdTradeSettlementCycle(merchant.getSettletype(), merchant.getDaily_timelystatus()));//交易结算周期
                        content.append("&&");
                        content.append("");//预留字段1
                        content.append("&&");
                        content.append("");//预留字段2
                        content.append("&&");
                        content.append("");//预留字段3
                        content.append("&&");
                        content.append("");//预留字段4

                        content.append("\n");//换行
                        bw.write(content.toString());
                        bw.flush();
                        id = Long.parseLong(String.valueOf(bean.getId()));
                    }
                    //最后一次跳出循环
                    if (count.intValue() < pageSize) {
                        flag = false;
                    }
                }
            }
            bw.close();
            LichuSystemPushUtil.taskNotify("下游", "生成浦发昨日交易文件", "浦发昨日交易文件生成成功，耗时：" + (DateUtils.getNow() - begintime) / 1000 + "s");
        } catch (IOException e) {
            logger.error("生成浦发昨日交易文件异常:",e);
            LichuSystemPushUtil.taskNotify("下游", "生成浦发昨日交易文件异常", "生成浦发昨日交易文件异常，请尽快处理，文件名：" + fileName);
        }

    }


    /**
     * 生成昨日的资金结算文件
     */
    public static void createSettlementFile(){

        File f = null;//新建文件
        SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd");//设置日期格式
        Calendar calendar = Calendar.getInstance();  //得到日历
        calendar.add(Calendar.DAY_OF_MONTH, -1);     //设置为昨天
        String yesterday = df.format(calendar.getTime());//得到昨天的的时间
        String localPath = "d:\\SPD_BANK\\"+yesterday+"\\";
        File path = new File(localPath);
        if (!path.exists()) {
            path.mkdirs();
        }
        String fileName ="12019900"+"_STLM_INFO_"+yesterday;//生成文件名 合作机构代码（四位）+总/分行号（四位）_TXN_INFO_日期
        String csvFilePath = "";
        csvFilePath = localPath + fileName;
        f = new File(csvFilePath);
        long begintime = DateUtils.getNow();
        try {
            if (f.exists()) {
                return;
            }
            f.createNewFile();
            //查询出商户表中需要上送的信息
            List<MerchantInfoSynOtherBean> merchants = merchantService.findMerchantNo(list_tags);
            if (merchants==null || merchants.size()<1){
                return;
            }
            BufferedWriter bw = new BufferedWriter(new FileWriter(f));
            for (MerchantInfoSynOtherBean merchant : merchants) {
                List<FundSettlementSynBean> detailSynBean = merchantService.findSettlementSynBean(yesterday,merchant.getMerchantid());
                if (detailSynBean == null || detailSynBean.size() < 1) {
                    continue;
                }

                for (FundSettlementSynBean bean : detailSynBean) {
                    StringBuffer content = new StringBuffer();
                    content.append(bean.getCredit_date());//入账日期
                    content.append("&&");
                    content.append(bean.getCredit_time());//入账时间
                    content.append("&&");
                    content.append(bean.getMerchantno());//内部商户号
                    content.append("&&");
                    content.append(merchant.getMerchantno_uni());//银联商户号
                    content.append("&&");
                    content.append(bean.getTotal_balance());//交易金额
                    content.append("&&");
                    content.append(bean.getSettlement_amount());//清算金额
                    content.append("&&");
                    content.append(bean.getTotal_fee());//应收商户手续费
                    content.append("&&");
                    content.append("");//手续费减免金额
                    content.append("&&");
                    content.append("");//预留金额1
                    content.append("&&");
                    content.append("");//预留金额2
                    content.append("&&");
                    content.append(merchant.getCardnum());//结算账号
                    content.append("&&");
                    content.append(spdSettlementType(merchant.getAccounttype()));//结算账号类型
                    content.append("&&");
                    content.append("");//备注
                    content.append("&&");
                    content.append("");//预留字段1
                    content.append("&&");
                    content.append("");//预留字段2
                    content.append("&&");
                    content.append("");//预留字段3
                    content.append("&&");
                    content.append("");//预留字段4

                    content.append("\n");//换行
                    bw.write(content.toString());
                    bw.flush();
                }
            }
            bw.close();
            LichuSystemPushUtil.taskNotify("下游", "生成浦发昨日资金结算文件", "浦发昨日资金结算文件生成成功，耗时：" + (DateUtils.getNow() - begintime) / 1000 + "s");
        } catch (IOException e) {
            logger.error("生成浦发昨日资金结算文件异常:",e);
            LichuSystemPushUtil.taskNotify("下游", "生成浦发昨日资金结算文件异常", "生成浦发昨日资金结算文件异常，请尽快处理，文件名：" + fileName);
        }
    }

    /**
     * 生成空文件
     */
    public static void createFinishFile(){

        try {
            SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd");//设置日期格式
            Calendar calendar = Calendar.getInstance();  //得到日历
            calendar.add(Calendar.DAY_OF_MONTH, -1);     //设置为昨天
            String yesterday = df.format(calendar.getTime());//得到昨天的的时间
            String localPath = "d:\\SPD_BANK_compression\\"+yesterday+"\\";
            File file = new File(localPath);
            if (!file.exists()){
                file.mkdirs();
            }

            String fileName ="12019900"+"_COMKT_"+yesterday+".finish";//生成文件名 例如：00019900_COMKT_20211101.finish
            String csvFilePath = "";
            csvFilePath = localPath + fileName;
            File f = new File(csvFilePath);//新建文件
            if (!f.exists()){
                f.createNewFile();
            }
        } catch (IOException e) {
            logger.error("创建空文件异常:",e);
        }

    }



    //将扫呗状态转为spd浦发状态
    public static String spdMerchantStatus(String saobei_status){
        //商户状态  商户是否有效
        //0-正常；6-冻结；8-注销  商户状态  0 正常  1新创建审核中 2二级商户银行卡信息更改审核中 3非二级商户支付配置更改审核中 4同步更新修改 5pos商户富友商户号修改  9审核驳回 10商户预审通过 98风控关停 99 关闭
        String status = saobei_status;
        if (StringUtils.equals(status,"97")||StringUtils.equals(status,"98")){
            status = "6";
        }else if(StringUtils.equals(status,"99")){
            status = "8";
        }else{
            status = "0";
        }
        return status;
    }

    //将扫呗的结算类型转为浦发的结算类型
    public static String spdSettlementType(String settlementtype){
        //结算类型 A-对公 P-对私 O-其他
        String accounttype = settlementtype;
        if (StringUtils.equals(accounttype,"1")){
            accounttype = "A";
        }else if (StringUtils.equals(accounttype,"2")){
            accounttype = "P";
        }else{
            accounttype = "O";
        }
        return accounttype;
    }

    //根据标签id,获取浦发分行号
    public static String spdSignLineNum(String tagid){
        String line_num = "";
        //签约分行号
        switch (tagid){
            case "1":
                line_num = "7000";
                break;
            case "2":
                line_num = "6400";
                break;
            case "3":
                line_num = "6600";
                break;
            case "4":
                line_num = "7300";
                break;
            case "5":
                line_num = "6400";
                break;
            default:
                line_num = "7600";
                break;
        }
        return line_num;
    }

    //扫呗交易类型转为浦发交易类型
    public static String spdTradeType(String pay_mode,String card_type, String trade_type){
        //浦发支付方式  01-借记卡 02-贷记卡 03-外卡 04-信用支付  11-微信 12-支付宝 62-银联扫码 99-其他
        //扫呗支付方式 1 微信 2支付宝 3银行卡 4 现金 5无卡支付 6qq钱包 7百度钱包8京东钱包 10翼支付 11云闪付
        if (StringUtils.equals(trade_type,"1")){
            trade_type = "11";
        }else if (StringUtils.equals(trade_type,"2")){
            trade_type = "12";
        }else if (StringUtils.equals(trade_type,"3")&&StringUtils.equals(card_type,"0")){
            trade_type = "01";
        }else if (StringUtils.equals(trade_type,"3")&&StringUtils.equals(card_type,"1")){
            trade_type = "02";
        }else if (StringUtils.equals(trade_type,"11")&&(StringUtils.equals(pay_mode,"1")||StringUtils.equals(pay_mode,"2"))){
            trade_type = "62";
        }else {
            trade_type = "99";
        }
        return trade_type;
    }


    //交易结算周期
    //0-T+0实时入账 1-T+1入账 2-D+1入账 9-其他
    public static String spdTradeSettlementCycle(String settletype,String daily_timelystatus ){
        String trade_settlement_cycle = "9";
        if (StringUtils.equals(settletype,"1")&&StringUtils.equals(daily_timelystatus,"0")){
            trade_settlement_cycle = "1";
        }
        if (StringUtils.equals(settletype,"1")&&StringUtils.equals(daily_timelystatus,"1")){
            trade_settlement_cycle = "2";
        }
        return trade_settlement_cycle;
    }

    //扫呗的交易类型转为浦发交易类型  CONS-消费；REVK-撤销；REFD-退货； AUTH-预授权完成； INST-信用支付；OTHE-其他
    //支付状态  支付成功1  支付失败2 支付中3  已撤销4  退款成功5  退款失败6
    public static String spdTradeType(String trade_type){
        String spdTradeType = "OTHE";
        if (StringUtils.equals(trade_type,"1") || StringUtils.equals(trade_type,"2") ||StringUtils.equals(trade_type,"3")  ){
            spdTradeType = "CONS";
        }else if (StringUtils.equals(trade_type,"4")){
            spdTradeType = "REVK";
        }else if (StringUtils.equals(trade_type,"5")||StringUtils.equals(trade_type,"6")){
            spdTradeType = "REFD";
        }else{
            spdTradeType = "OTHE";
        }
        return spdTradeType;
    }

    //判断是否为撤销或退款  如果是则返回对应的原商户订单号 原系统流水号  否则为""
    public static String spdOut_Refund_no(String spdTradeType ,String out_trade_no){
        String spdOut_trade_no = "";
        if (StringUtils.equals(spdTradeType,"REVK")||StringUtils.equals(spdTradeType,"REFD")){
            spdOut_trade_no = out_trade_no;
        }
        return spdOut_trade_no;
    }



    //压缩tar.gz,ftp上传浦发
    public static void fileToZip(String yesterday) {
        ZipOutputStream out = null;
        String sourceFilePath = "d:\\SPD_BANK\\"+yesterday+"\\";
        String zipFilePath = "d:\\SPD_BANK_compression"+File.separator+yesterday + File.separator +"12019900_COMKT_"+ yesterday + ".tar.gz";
        String path = "d:\\SPD_BANK_compression"+File.separator+yesterday + File.separator;
        try {
            File file = new File(path);
            if (!file.exists()){
                file.mkdirs();
            }
            File zipFile = new File(zipFilePath);

            out = new ZipOutputStream(new FileOutputStream(zipFile));

            File sourceFile = new File(sourceFilePath);//判断这个路径下面是否有文件
            if(sourceFile.exists() == false) {
                logger.info("待压缩的文件目录：" + sourceFilePath + " 不存在.");
                if (out != null) {
                    out.flush();
                    out.close();
                }
                return;
            }
            if(sourceFile.isFile()) {//判断是否是文件
                zipFileOrDirectory(out,sourceFile,"");
            }else{
                File[] sourceFiles = sourceFile.listFiles();
                for(int i=0;i<sourceFiles.length;i++){
                    // 递归压缩，更新curPaths
                    zipFileOrDirectory(out,sourceFiles[i],"");
                }
            }
            if (out != null) {
                out.flush();
                out.close();
            }
            //删除本地文件
            deleteFile(sourceFile);
//            deleteFile(zipFile);
        } catch (Exception e) {
            if (out != null) {
                try {
                    out.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
            }
            e.printStackTrace();
        }
    }

    private static void zipFileOrDirectory(ZipOutputStream out,File fileOrDirectory, String curPath) throws IOException {
        FileInputStream in = null;
        try{
            if (!fileOrDirectory.isDirectory()) {
                // 压缩文件
                byte[] buffer = new byte[4096];
                int bytes_read;
                in = new FileInputStream(fileOrDirectory);
                ZipEntry entry = new ZipEntry(curPath + fileOrDirectory.getName());
                out.putNextEntry(entry);
                while ((bytes_read = in.read(buffer)) != -1){
                    out.write(buffer, 0, bytes_read);
                }
                out.closeEntry();
            }else{
                //压缩目录
                File[] entries = fileOrDirectory.listFiles();
                for (int i = 0; i < entries.length; i++) {
                    // 递归压缩，更新curPaths
                    zipFileOrDirectory(out, entries[i], curPath + fileOrDirectory.getName() + "/");
                }
            }
        } catch (IOException ex) {
            ex.printStackTrace();
            throw ex;
        } finally {
            if (in != null) {
                in.close();
            }
        }
    }

    //删除本地文件
    public static void deleteFile(File file) {
        if (file.exists()) {//判断文件是否存在
            if(file.isFile()){//判断是否是文件
                file.delete();//删除文件
            } else if(file.isDirectory()) {//否则如果它是一个目录 
                File[] files = file.listFiles();//目录下所有的文件
                for(int i = 0;i < files.length;i ++) {//遍历目录下所有的文件
                    deleteFile(files[i]);//把每个文件用这个方法进行迭代删除
                }
//                file.delete();//删除文件夹
            }
        }else{
            logger.info("所删除的文件不存在");
        }
    }

}
~~~



#### 从数据库查询出数据

service类

~~~java
	/**
	 * 查询商户表中需要上送的信息
	 * @param tagList  标签id列表
	 * @return
	 */
	public List<MerchantInfoSynOtherBean> findMerchantNo(List<String> tagList){
		return merchantDao.findMerchantNo(tagList);
	}

	/**
	 * 查询浦发商户信息上送信息
	 * @param tagList  标签id列表
	 * @return
	 */
	public List<MerchantInfoSynBean> findInfoSynBean(List<String> tagList){
		return merchantDao.findInfoSynBean(tagList);
	}

	/**
	 * 查询浦发商户交易明细上送信息
	 * @param yesterday  昨天时间
	 * @param merchantid 商户id
	 * @return
	 */
	public List<TransactionDetailSynBean> findDetailSynBean(String yesterday,String merchantid,long id,int pageSize){
		return merchantDao.findDetailSynBean(yesterday,merchantid,id,pageSize);
	}

	/**
	 * 查询浦发商户资金结算上送信息
	 * @param yesterday  昨天时间
	 * @param merchantid 商户id
	 * @return
	 */
	public List<FundSettlementSynBean> findSettlementSynBean(String yesterday,String merchantid){
		return merchantDao.findSettlementSynBean(yesterday,merchantid);
	}
~~~



dao类

~~~java
	
	List<MerchantInfoSynOtherBean> findMerchantNo(List<String> tagList);

	List<MerchantInfoSynBean> findInfoSynBean(List<String> tagList);

	List<TransactionDetailSynBean> findDetailSynBean(String yesterday,String merchantid,long id,int pageSize);

	List<FundSettlementSynBean> findSettlementSynBean(String yesterday,String merchantid);
~~~



Impl类

这里数据库数据联查，数据较多，都采用单表去查，然后利用查出的数据再去查询。

~~~java
/**查询商户表中需要上送的信息
	 * SQL:select m.id,m.settletype ,m.daily_timelystatus,m.merchantno_uni,m.cardnum,m.accounttype FROM b_merchant_next m LEFT JOIN f_user_tag f ON f.fid = m.id  WHERE 1 = 1  and f.rolesid = 5 AND f.tagid IN ( 1, 2, 3, 4, 5 ) and m.status<99
	 * EXPLAIN:
	 * 1	SIMPLE	f	ALL	fid				23979	Using where
	 * 1	SIMPLE	m	eq_ref	PRIMARY,idx_status	PRIMARY	4	te5.f.fid	1	Using where
	 * 耗时: 0.276s
	 * 数据源:te5
	 * @param tagList  标签id列表
	 * @return
	 */
	List<MerchantInfoSynOtherBean> findMerchantNo(List<String> tagList){
		List<MerchantInfoSynOtherBean> list =new ArrayList<>();
		String sql="select m.id,m.settletype ,m.daily_timelystatus,m.merchantno_uni,m.cardnum,m.accounttype " +
				"FROM b_merchant_next m LEFT JOIN f_user_tag f ON f.fid = m.id " +
				"WHERE f.rolesid = 5 AND f.tagid  IN :tagList and m.status<99";

		Query query =em.createNativeQuery(sql);
		query.setParameter("tagList", tagList);
		List result = query.getResultList();
		for (Object object : result) {
			MerchantInfoSynOtherBean baseBean = new MerchantInfoSynOtherBean();
			Object[] row = (Object[])object;
			//对应的信息
			baseBean.setMerchantid(row[0]!=null?row[0].toString():"");
			baseBean.setSettletype(row[1]!=null?row[1].toString():"");
			baseBean.setDaily_timelystatus(row[2]!=null?row[2].toString():"");
			baseBean.setMerchantno_uni(row[3]!=null?row[3].toString():"");
			baseBean.setCardnum(row[4]!=null?row[4].toString():"");
			baseBean.setAccounttype(row[5]!=null?row[5].toString():"");
			list.add(baseBean);
		}

		return list;
	}

	/**查询浦发商户信息上送信息
	 * SQL:SELECT m.merchantno, m.merchantno_uni, m.STATUS, m.merchantname, m.merchantalias, m.license_no, m.cardnum, m.accounttype, m.cardname, m.STATUS, m.STATUS, m.merchantperson, m.merchantphone, m.merchantprovince, m.merchantcity, m.merchantaddress, m.STATUS, m.STATUS, m.merchantwx3id,f.tagid
	 * FROM b_merchant_next m LEFT JOIN f_user_tag f ON f.fid = m.id
	 * WHERE 1 = 1  and f.rolesid = 5  AND f.tagid IN ( 1, 2, 3, 4, 5 ) and `status`<99
	 * EXPLAIN:
	 * 1	SIMPLE	f	ALL	fid				23979	Using where
	 * 1	SIMPLE	m	eq_ref	PRIMARY,idx_status	PRIMARY	4	te5.f.fid	1	Using where
	 * 耗时: 0.393s
	 * 数据源:te5
	 * @param tagList  标签id列表
	 * @return
	 */
	List<MerchantInfoSynBean> findInfoSynBean(List<String> tagList) throws Exception {
		List<MerchantInfoSynBean> list =new ArrayList<>();

		String sql = "SELECT m.merchantno, m.merchantno_uni, m.STATUS,m.merchantcompany, m.merchantname, m.license_no," +//商户号 银联商户号 商户状态 商户法定名称 商户经营名称 营业证明文件号码
				" m.cardnum, m.accounttype, m.cardname, m.merchantperson, m.merchantphone, " +//结算账号	结算账号类型 结算账号户名 商户联系人姓名 商户联系电话
				"m.merchantprovince, m.merchantcity, m.merchantaddress,m.merchantwx3id,f.tagid,  " +//商户所在省  商户所在市  商户地址  商户MCC 标签id
				"m.platform_debitcard_top_fee " +//平台储蓄卡封顶费用（分）
				"FROM b_merchant_next m LEFT JOIN f_user_tag f ON f.fid = m.id " +
				"WHERE f.rolesid = 5 AND f.tagid  IN :tagList and m.status<99 ";
		Query query =em.createNativeQuery(sql);
		query.setParameter("tagList", tagList);

		List result = query.getResultList();
		for (Object object : result) {
			MerchantInfoSynBean baseBean = new MerchantInfoSynBean();
			Object[] row = (Object[])object;
			//对应的信息
			//Row1
			baseBean.setMerchantno(row[0]!=null?row[0].toString():"");
			baseBean.setMerchantno_uni(row[1]!=null?row[1].toString():"");
			baseBean.setStatus(row[2]!=null?row[2].toString():"");
			baseBean.setMerchantcompany(row[3]!=null?row[3].toString():"");
			baseBean.setMerchantname(row[4]!=null?row[4].toString():"");
			baseBean.setLicense_no(row[5]!=null?row[5].toString():"");
			//Row2
			baseBean.setCardnum(row[6]!=null?row[6].toString():"");
			baseBean.setAccounttype(row[7]!=null?row[7].toString():"");
			baseBean.setCardname(row[8]!=null?row[8].toString():"");
			baseBean.setMerchantperson(row[9]!=null?row[9].toString():"");
			baseBean.setMerchantphone(row[10]!=null?row[10].toString():"");
			//Row3
			baseBean.setProvinceAndCity((row[11]!=null?row[11].toString():"") + (row[12]!=null?row[12].toString():""));
			baseBean.setMerchantaddress(row[13]!=null?row[13].toString():"");
			baseBean.setMerchantwx3id(row[14]!=null?row[14].toString():"");
			baseBean.setSignLineNum((row[15] != null) ? row[15].toString() : "");
			String pos_borrow_value = (row[16] != null) ? row[16].toString() : "0";
			if (StringUtils.equals(pos_borrow_value,"0")){
				pos_borrow_value = NumberUtil.div(pos_borrow_value,"100",2);
			}
			baseBean.setPos_borrow_value(pos_borrow_value);


			list.add(baseBean);
		}

		return list;
	}

	/**查询浦发商户交易明细上送信息
	 * SQL:SELECT a.createdate, a.createtime, a.settle_date, a.merchant_no, a.uni_merchant_no, a.terminal_id, a.type, a.uni_card_no, a.total_fee, a.refund_fee, a.balance, a.poundage, a.terminal_trace, a.channel_trade_no
	 * FROM t_tradeserial_main a
	 * WHERE 1 = 1 AND  a.settle_date = '20211130' and a.merchantid = '1662930'
	 * EXPLAIN:
	 * 1	SIMPLE	a	ref	merchantid	merchantid	5	const	66	Using where
	 * 耗时: 0.067s
	 * 数据源:te5
	 * @param yesterday  昨天时间
	 * @param merchantid 商户id
	 * @return
	 */
	List<TransactionDetailSynBean> findDetailSynBean(String yesterday,String merchantid,long id,int pageSize){
		List<TransactionDetailSynBean> list =new ArrayList<>();
		String sql = "select a.createdate,a.createtime,a.settle_date,a.merchant_no,a.uni_merchant_no,a.terminal_id," +//交易日期 交易时间 清算日期 商户号 银联商户号 终端号
				"a.type,a.uni_card_no,a.total_fee,a.refund_fee,a.balance,a.terminal_trace,a.channel_trade_no,a.order_body, " +//支付方式 卡号 交易金额 退款金额 清算金额 终端流水号 第三方订单号 订单描述
				"a.out_trade_no ,a.pay_mode,a.card_type,a.pay_status_code,a.out_refund_no,a.id "+//订单号 支付类型 银行卡类型 支付状态 退款单号 id
				"from t_tradeserial_main a " +
				"WHERE  a.settle_date = :settle_date and a.merchantid = :merchantid ";
		String limitPart = "";
		if(id == 0L){//第一页查询
			limitPart = "limit "+pageSize;
		}else{
			sql +=" and id >"+id;
			limitPart = " limit "+pageSize;
		}
		Query query =em.createNativeQuery(sql+limitPart);
		if (yesterday!=null){
			query.setParameter("settle_date", yesterday);
		}
		if (merchantid!=null){
			query.setParameter("merchantid", merchantid);
		}else {
			query.setParameter("merchantid", "0");
		}



		List result = query.getResultList();
		for (Object object : result) {
			TransactionDetailSynBean baseBean = new TransactionDetailSynBean();
			Object[] row = (Object[])object;
			//对应的信息
			//Row1
			String trade_date = row[0]!=null?row[0].toString():"";
			if (!StringUtils.equals(trade_date,"")){
				trade_date = trade_date.replaceAll("-","");
			}
			baseBean.setTrade_date(trade_date);

			String trade_time = row[1]!=null?row[1].toString():"";
			if (!StringUtils.equals(trade_time,"")){
				trade_time = trade_time.substring(11, 19).replaceAll(":","");
			}
			baseBean.setTrade_time(trade_time);

			String settlement_date = row[2]!=null?row[2].toString():"";
			if (!StringUtils.equals(settlement_date,"")){
				settlement_date = settlement_date.replaceAll("-","");
			}
			baseBean.setSettlement_date(settlement_date);

			baseBean.setMerchantno(row[3]!=null?row[3].toString():"");
			baseBean.setMerchantno_uni(row[4]!=null?row[4].toString():"");
			baseBean.setTerminal_id(row[5]!=null?row[5].toString():"");
			//Row2
			baseBean.setPay_type(row[6]!=null?row[6].toString():"");
			baseBean.setTrade_cardnum(row[7]!=null?row[7].toString():"");
			baseBean.setTotal_fee(row[8]!=null?row[8].toString():"");
			baseBean.setRefund_fee(row[9]!=null?row[9].toString():"");
			baseBean.setBalance(row[10]!=null?row[10].toString():"");
			baseBean.setTerminal_trace_no(row[11]!=null?row[11].toString():"");
			baseBean.setChannel_trade_no(row[12]!=null?row[12].toString():"");
			baseBean.setRemarks(row[13]!=null?row[13].toString():"");
			//ROW3
			baseBean.setOut_trade_no(row[14]!=null?row[14].toString():"");
			baseBean.setPay_mode(row[15]!=null?row[15].toString():"");
			baseBean.setCard_type(row[16]!=null?row[16].toString():"");
			baseBean.setTrade_type(row[17]!=null?row[17].toString():"");
			baseBean.setOut_refund_no(row[18]!=null?row[18].toString():"");
			baseBean.setId(StringUtil.ObjtoInt(row[19]));


			list.add(baseBean);
		}

		return list;
	}

	/**查询浦发商户资金结算上送信息
	 * SQL:SELECT s.createtime,s.merchantno,s.total_balance,s.total_fee FROM s_merchant_day s WHERE 1 = 1 and s.tradedate = '20211130' and s.merchantid = '13376'
	 * EXPLAIN:
	 * 1	SIMPLE	s	ref	tradedate,merchantid	merchantid	4	const	5	Using where
	 * 耗时: 0.243s
	 * 数据源:te5
	 * @param yesterday  昨天时间
	 * @param merchantid 商户id
	 * @return
	 */
	List<FundSettlementSynBean> findSettlementSynBean(String yesterday,String merchantid){
		List<FundSettlementSynBean> list =new ArrayList<>();
		String sql = "SELECT s.createtime,s.merchantno,s.total_balance,s.total_fee " +//创建时间  商户号  交易金额  应收商户手续费
				"FROM s_merchant_day s " +
				"WHERE s.tradedate = :tradedate and s.merchantid = :merchantid";
		Query query =em.createNativeQuery(sql);
		if (yesterday!=null){
			query.setParameter("tradedate", yesterday);
		}
		if (merchantid!=null){
			query.setParameter("merchantid", merchantid);
		}else {
			query.setParameter("merchantid", "0");
		}

		List result = query.getResultList();
		for (Object object : result) {
			FundSettlementSynBean baseBean = new FundSettlementSynBean();
			Object[] row = (Object[])object;
			//对应的信息
			String createtimes = row[0]!=null?row[0].toString():"";
			String createtime = "";
			String createdate = "";
			if (!StringUtils.equals(createtimes,"")){
				createdate = createtimes.substring(0, 10).replaceAll("-","");
				createtime = createtimes.substring(11, 19).replaceAll(":","");
			}
			//入账日期  入账时间   商户号    交易金额  清算金额  应收商户手续费
			baseBean.setCredit_date(createdate);
			baseBean.setCredit_time(createtime);
			baseBean.setMerchantno(row[1]!=null?row[1].toString():"");
			baseBean.setTotal_balance(row[2]!=null?row[2].toString():"");

			BigDecimal settlement_amount;
			BigDecimal total_balance = new BigDecimal(row[2]!=null?row[2].toString():"0");
			BigDecimal total_fee = new BigDecimal(row[3]!=null?row[3].toString():"0");
			settlement_amount = total_balance.subtract(total_fee);
			baseBean.setSettlement_amount(settlement_amount.toString());

			baseBean.setTotal_fee(row[3]!=null?row[3].toString():"");

			list.add(baseBean);
		}

		return list;
	}
~~~





#### sftp工具类新增方法

~~~java
 /**
     * 批量上传文件
     * @param yesterday 昨天日期
     * @return
     * @throws SftpException
     */
    public static void bacthUploadSPDFile(String yesterday) throws IOException, SftpException {
        String localPath = "d:\\SPD_BANK_compression" + File.separator + yesterday + File.separator;//本地上传目录(以路径符号结束)
        String directory = "/upload/spdbfile/lichusftp";//SftpUtil.getSftp_path();//SFTP上传路径  /upload/spdbfile/lichusftp
        //构造基于密码认证的sftp对象
        SftpUtil sftpUtil = new SftpUtil("lcswsftp", "lcswsftp123", "192.168.1.12", 23);
        sftpUtil.login();
        sftpUtil.cdDirectory(directory);
        try {
            File file = new File(localPath);
            File[] files = file.listFiles();
            for (int i = 0; i < files.length; i++){
                if (files[i].isFile()){
//	        		uploadFile( hbwkno, files[i].getName(),sftpUtil);
                    sftpUtil.uploadSPDFile( yesterday, files[i].getName());
                }
            }
            //删除本地文件
//            deleteFile(file);
//	        sftp.logout();
        } catch (Exception e) {
            log.error("浦发商户批量上传异常:"+e.toString());
            e.printStackTrace();
        }finally{
            sftpUtil.logout();
        }
    }
    /**
     * 上传单个文件
     * @param remotePath：远程保存目录
     * @param remoteFileName：保存文件名
     * @param localPath：本地上传目录(以路径符号结束)
     * @param localFileName：上传的文件名
     * @return
     */
    public static void uploadSPDFile(String yesterday,String fileName) throws IOException {
        String remotePath = "/upload/spdbfile/lichusftp";
        String filePath = "d:\\SPD_BANK_compression" + File.separator + yesterday + File.separator + fileName;
        //构造基于密码认证的sftp对象
        SftpUtil sftp = new SftpUtil("lcswsftp", "lcswsftp123", "192.168.1.12", 23);
        sftp.login();
        InputStream in = null;
        try {
            in = new FileInputStream(new File(filePath));
            //上传到该目录  ,sftp端文件名,输入流
            sftp.uploadSPD(remotePath, fileName, in);
        } catch (Exception e) {
            log.error("浦发商户上传sftp异常:"+e.toString());
            e.printStackTrace();
        }finally{
            if (in != null) {
                in.close();
            }
            sftp.logout();
        }
    }

    /**
     * 将输入流的数据上传到sftp作为文件  
     *
     * @param directory
     *            上传到该目录  
     * @param sftpFileName
     *            sftp端文件名  
     * @param input
     *            输入流  
     * @throws SftpException
     * @throws Exception
     */
    public void uploadSPD(String directory, String sftpFileName, InputStream input) throws SftpException{
        try {
            sftp.cd(directory);
        } catch (SftpException e) {
            log.warn("directory is not exist");
            String[] split = directory.split("/");
            mkdirDir(split,"",split.length,0);
        }
        sftp.put(input, sftpFileName);
        log.info("file:{} is upload successful" , sftpFileName);
    }

    /**
     * 递归根据路径创建文件夹
     *
     * @param dirs     根据 / 分隔后的数组文件夹名称
     * @param tempPath 拼接路径
     * @param length   文件夹的格式
     * @param index    数组下标
     * @return
     */
    public void mkdirDir(String[] dirs, String tempPath, int length, int index) {
        // 以"/a/b/c/d"为例按"/"分隔后,第0位是"";顾下标从1开始
        index++;
        if (index < length) {
            // 目录不存在，则创建文件夹
            tempPath += "/" + dirs[index];
        }
        try {
            sftp.cd(tempPath);
            if (index < length) {
                mkdirDir(dirs, tempPath, length, index);
            }
        } catch (SftpException ex) {
            try {
                sftp.mkdir(tempPath);
                sftp.cd(tempPath);
            } catch (SftpException e) {
                log.info("进入sftp文件夹异常:",e);

            }
            mkdirDir(dirs, tempPath, length, index);
        }
    }
~~~



