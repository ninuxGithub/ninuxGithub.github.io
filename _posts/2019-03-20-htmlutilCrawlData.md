---
title: htmlutil爬数据
author: ninuxGithub
layout: post
date: 2019-3-21 13:05:52
description: "htmlutil爬数据"
tag: htmlutil
---


### 目的
    
    前端项目通过js调用公共接口获取数据，然后经过一系列的规则进行数据的过滤，形成一个最终的结果
    
    需求： 对股票一键选进行历史数据的保存
    
    解决方案： 采用htmlutil进行数据的爬取， 模仿鼠标点击页面， 等待js执行接口完毕，渲染到页面后， 获取页面dom元素的值
    
    
### demo 代码

```java
package com.example.api.crawler;

import com.gargoylesoftware.htmlunit.BrowserVersion;
import com.gargoylesoftware.htmlunit.NicelyResynchronizingAjaxController;
import com.gargoylesoftware.htmlunit.WebClient;
import com.gargoylesoftware.htmlunit.html.HtmlElement;
import com.gargoylesoftware.htmlunit.html.HtmlPage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.*;

//对日志进行控制 ： 禁止 

//            LogFactory.getFactory().setAttribute("org.apache.commons.logging.Log", "org.apache.commons.logging.impl.NoOpLog");
//            java.util.logging.Logger.getLogger("httpclient.wire").setLevel(Level.OFF);
//            java.util.logging.Logger.getLogger("com.gargoylesoftware.htmlunit").setLevel(Level.OFF);
//            java.util.logging.Logger.getLogger("org.apache.commons.httpclient").setLevel(Level.OFF);
//            java.util.logging.Logger.getLogger("org.apache.http").setLevel(Level.OFF);

/**
 * @author shenzm
 * @date 2019-3-20
 * @description 作用
 */
public class OneKeySelectUpdate {


    private static final Logger logger = LoggerFactory.getLogger(OneKeySelectUpdate.class);

    public static void main(String[] args) {
        for (int i=0; i<5; i++){
            scrawData();
        }


    }

    private static void scrawData() {
        logger.info("开始爬数据");
        Map<Integer, List<Integer>> indexMap = new HashMap<>();
        indexMap.put(1, Arrays.asList(1, 2, 3, 4, 5));
        indexMap.put(2, Arrays.asList(6, 7, 8, 9, 10, 11));
        indexMap.put(3, Arrays.asList(12, 13, 14, 15, 16));
        Map<String, Map<String, List<String>>> dataMap = new HashMap<>();

        int len = 16;
        for (int i = 1; i <= len; i++) {
            extractRowMap(i, dataMap, indexMap);
        }

        for (String key : dataMap.keySet()) {
            for (String liKey : dataMap.get(key).keySet()) {
                logger.info(key + "-->" + liKey + "-->" + dataMap.get(key).get(liKey));
            }
        }
    }

    private static void extractRowMap(int k, Map<String, Map<String, List<String>>> dataMap, Map<Integer, List<Integer>> indexMap) {
        WebClient webClient = null;
        try {
            webClient = new WebClient(BrowserVersion.CHROME);
            webClient.getOptions().setJavaScriptEnabled(true);
            webClient.setAjaxController(new NicelyResynchronizingAjaxController());
            webClient.getOptions().setCssEnabled(true);
            webClient.getOptions().setRedirectEnabled(true);
            webClient.getOptions().setThrowExceptionOnScriptError(false);
            webClient.getOptions().setTimeout(8000);
            webClient.setJavaScriptTimeout(8000);
            HtmlPage page = (HtmlPage) webClient.getPage("https://xxxxxxx/index.html");
            webClient.waitForBackgroundJavaScript(10000);
            List<HtmlElement> menuList = page.getByXPath("//section[@class='nav_box']/span");

            int tabIndex = 0;
            int rowIndex = 0;
            outer:
            for (Integer ti : indexMap.keySet()) {
                List<Integer> list = indexMap.get(ti);
                inner:
                for (int index = 0; index < list.size(); index++) {
                    if (list.get(index) == k) {
                        tabIndex = ti;
                        rowIndex = index;
                        break outer;
                    }
                }
            }
            HtmlElement menuTabHtml = menuList.get(tabIndex);

            //获取首页的tab的类型热门，短线，淘金三个类型
            String tabName = menuTabHtml.asText();
            Map<String, List<String>> menuMap = dataMap.get(tabName);
            if (menuMap == null) {
                menuMap = new HashMap<>();
            }

            HtmlPage clickTabPage = menuTabHtml.click();
            List<HtmlElement> liList = clickTabPage.getByXPath("//ul[@class='model_box']/li");
            HtmlElement rowHtmlElement = liList.get(rowIndex);
            List<HtmlElement> liRows = rowHtmlElement.getByXPath("//h3");
            // 每个tab类型下面拥有的li
            String rowText = liRows.get(rowIndex).asText();
            final HtmlElement htmlElement = liList.get(rowIndex);
            List<String> list = menuMap.get(rowText);
            if (list == null) {
                list = new ArrayList<>();
            }
            HtmlPage liPage = htmlElement.click();
            List<HtmlElement> domList = liPage.getByXPath("//li[@class='share_list']/div/span[@class='num']");
            for (HtmlElement dom : domList) {
                list.add(dom.asText());
            }
            menuMap.put(rowText, list);
            dataMap.put(tabName, menuMap);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (webClient != null) {
                webClient.close();
            }
        }
    }

}

```


```json
淘金-->高送转预期-->[300762, 002243, 603018, 603659, 002916, 603136, 002859, 603337, 002937, 300722, 000661, 600449, 600702, 300751]
淘金-->小而美-->[600250, 002803, 600262, 300047, 300684, 000615, 000785, 603903, 300427, 000892, 002637, 300423, 000759, 000606, 600793, 300272, 600981, 002159, 600985, 300583, 300461, 300513, 000922, 600738, 300467, 300391, 600387, 603518, 300568, 002208, 600802, 603067, 300107, 600609, 300632, 300389, 300422, 000821, 603357, 300761, 000537, 300506, 603086, 603330, 300437, 000736, 300637, 600097, 002798, 300571, 002034, 002182]
淘金-->高速成长-->[603018, 600603, 002228, 600681, 002258]
淘金-->稳定盈利-->[000877, 603018, 600603, 603337, 000723, 300285, 600167, 601155, 000401, 002075, 601898, 000898, 000055, 600567, 300132, 603027, 002258, 300193, 601015, 002442, 600702]
淘金-->白马股-->[603018, 002803, 002589, 000062, 600406, 300398, 300684, 300176, 603811, 002614, 300668, 603108, 603387, 601360, 600585, 000002, 002458, 600639, 000656, 300470, 300423, 002853, 000606, 002842, 600793, 603568, 002859, 000636, 300178, 002717, 002146, 002396, 603337, 300395, 300316, 600985, 300461, 600645, 300452, 300657, 300285, 300476, 002061, 600383, 601155, 002882, 002741, 600803, 000042, 002753, 000401, 000922, 603799, 002626, 603639, 300207, 601888, 300203, 300618, 002318, 600566, 002001, 600387, 300554, 603260, 002640, 000029, 603399, 600801, 600346, 002391, 603388, 300401, 603663, 300388, 603638, 002867, 603360, 300003, 600771, 600516, 300632, 603916, 300389, 002110, 300408, 603669, 000049, 300422, 000821, 600779, 002038, 603357, 300761, 002768, 000789, 600031, 600332, 601100, 000537, 002258, 300349, 603929, 300506, 603086, 000858, 300437, 000671, 000651, 000736, 600326, 000830, 002120, 600426, 002234, 603113, 603737, 600596, 600702, 300727, 002798, 600486, 000546, 002832, 600665, 603369, 600809, 300409, 002233, 002833]
热门-->市场关注-->[600928, 600624, 600604, 002195, 000750, 300059, 002600, 600030, 601099, 600776, 601066, 600518, 000725, 601319, 000063, 601318, 600519, 000858, 600050, 601519]
热门-->新股聚焦-->[600928, 002950, 300763, 300762, 002951, 002945, 603121, 601298, 601860, 601598, 002948, 300755, 603332, 603629, 300758, 603956, 300756, 603351, 601975, 601865, 002947, 603700, 300761, 603739, 603185, 300759, 601615, 002946, 300757, 002949]
热门-->一周明星-->[000993, 002950, 002356, 000159, 300763, 300762, 002951, 600624, 600156, 300471]
热门-->主力追踪-->[600268, 600621, 002356, 600928, 002017, 002302, 000990, 000066, 000159, 600624, 600536, 000563, 600895, 000877, 300471, 600635, 002195, 000750, 600030, 002535, 600518, 000725, 600048, 601318, 600031, 600519, 000858, 000651, 002027, 000860]
热门-->今日涨停股-->[600108, 000506, 601212, 000677, 000721, 600509, 603169, 600268, 600232, 603798, 600831, 000905, 002307, 000993, 000570, 600463, 600037, 002824, 600621, 000788, 002750, 002137, 603569, 002488, 002356, 002673, 600928, 300663, 300334, 300682, 002398, 002950, 300165, 000156, 300536, 600158, 600679, 600345, 300103, 002302, 000037, 603888, 600131, 603000, 002017, 600783, 002578, 300763, 300337, 000990, 300080, 600800, 000159, 000503, 600846, 000504, 600850, 002189, 600530, 600088, 000066, 300236, 000526, 300125, 300380, 600730, 300414, 300762, 002565, 002639, 002951, 000610, 600695, 603315, 000021, 600075, 600624, 002473, 000826, 601515, 300011, 600722, 002683, 600072, 600493, 002109, 600734, 600620, 000590, 601226, 600226, 600359, 600163]
短线-->出水芙蓉-->[]
短线-->底部红三兵-->[000936]
短线-->多方炮-->[000526, 000021, 600536, 600410, 600830, 600637, 600456, 300525, 002708, 300182, 002274, 600064, 000915, 300036, 000790, 300512, 600640, 300127, 300390, 300582, 601900, 300580, 002368, 603766, 600794, 600406, 002643, 601368, 002104, 002275, 000676, 300681, 603977, 600883, 600353]
短线-->身怀六甲-->[600150, 002520, 002825, 002372, 002158, 600545, 600751, 603698, 002865, 000613, 300538, 600992, 300116, 000701, 603985, 600097, 002805, 002242, 000888, 300024, 300702, 603676, 603963, 601588, 300106, 002182, 300603]
短线-->上升三步曲-->[600386]
短线-->曙光初现-->[000021, 300525, 002348, 600218, 600086, 002856, 300016]
```