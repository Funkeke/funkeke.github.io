---
title: Hello World 2024
---

兜兜转转一大圈，终究还是回到了原点，记录数据今天 20240312 迁移到这里

```python
print("hello world")
```

```php
<?php
if ('cli' != php_sapi_name()) {
    echo("ERROR: This program is for command line mode only.\n");
    exit(1);
}

chdir(dirname(__FILE__));
require('../../.root.php');
require(WEBAPP_ROOT);

using('callcenter_isp_action');
using('CallCenter.GnwayCallCenter');
using('DingDingRobot');

/**
 * Class CheckMissedRecord_page
 *  检查当天服务DB记录和ES是否一致
 */
// */5 * * * * php /u01/gnway/www/bangwo8.com/osp2016/callcenter/command/cron/ServiceIdleStateCheck.php

class ServiceIdleStateCheck_page
{
    protected $dbObj;
    protected $pageObj;
    protected $callCenterObj;
    protected $callCenterIspAct;

    public function __construct(&$dbObj, &$pageObj)
    {
        $this->dbObj = $dbObj;
        $this->pageObj = $pageObj;
        $this->callCenterIspAct = new tf_callcenter_isp_action($this->dbObj, $this->pageObj);
        $this->callCenterAccountObj = new callCenterAccountData($this->dbObj, $this->pageObj);
    }

    public function process()
    {
        $aids = MConfig::$current->cc->idlestate->aids ?: '';
//        $aids = "598420,765186,755178";
        $idle_state_notice_limit = MConfig::$current->cc->idlestate->notice_limit ?: 6;
        $idle_state_min_time = MConfig::$current->cc->idlestate->min_time ?: 10; // 需要配置
        $idle_state_max_time = MConfig::$current->cc->idlestate->max_time ?: 200;

        if (empty($aids)) exit(0);
        $aids_array = explode(',', $aids);

        $ten = $five = [];
        foreach ($aids_array as $aid) {
            $serviceListInfo = $this->callCenterIspAct->getCallServicerListInfoByAId($aid);
            $serviceList = $serviceListInfo['servicerList'] ?: [];
            unset($serviceListInfo);
            if (empty($serviceList)) continue;

            foreach ($serviceList as $sid) {
                // 最后服务时间
                $detail = $this->callCenterAccountObj->getCallServicerDetailState($sid,[
                    'sId',
                    'passportName',
                    'mobile',
                    'state',
                    'voipStatus',
                    'idleState',
                    'channelUuid',
                    'currCallId',
                    'channelTime',
                    'lastServiceEndTime'
                ]);
                if (!$detail['idleState']) continue;

                // 忙碌状态持续5分钟
                $diff = time() - ($detail['channelTime'] ? strtotime($detail['channelTime']) : $detail['lastServiceEndTime']);
                $detail['lastServiceEndTime'] = date('Y-m-d H:i:s', $detail['lastServiceEndTime']);

                // 忙碌状态持续大于5分钟，小于10分钟
                if ($diff > $idle_state_min_time && $diff< $idle_state_max_time) $five[$sid] = $detail;

                // 忙碌状态持续10分钟
                $diff > $idle_state_max_time && $ten[$sid] = $detail;
            }
        }
        $env = defined("PRIVATE_RUN")
            ? "【private-".PRIVATE_RUN."-".GNWAY_TONGFU_VERSION."】"
            : "【saas-".GNWAY_TONGFU_VERSION."】";

        $title = $content = "### 通话状态异常预警".$env.":".PHP_EOL;
        $content.= "* 检测时间：".date('Y-m-d H:i:s').PHP_EOL;

        $idle_state_max_minute = ceil($idle_state_max_time / 60);
        $idle_state_min_minute = ceil($idle_state_min_time / 60);

        $need_send = false;

        $content.= "* 忙碌状态持续{$idle_state_min_minute}>min且<{$idle_state_max_minute}min".PHP_EOL;
        $request_id = uniqid('idlestate_notice_');
        if (!empty($five)) {
            if (count($five) < $idle_state_notice_limit) {
                $content.="> ".json_encode($five, JSON_UNESCAPED_UNICODE);
            } else {
                $log_data['five_min'] = $ten;
                $content.= "> 超过{$idle_state_notice_limit}个，请查看日志 RequestId: ".$request_id.PHP_EOL;
            }
            $need_send = true;
        } else {
            $content.="> 暂无数据".PHP_EOL;
        }
        $content.=PHP_EOL;
        $content.= "* 忙碌状态持续>{$idle_state_max_minute}min".PHP_EOL;
        if (!empty($ten)) {
            if (count($ten) < $idle_state_notice_limit) {
                $content.="> ".json_encode($ten, JSON_UNESCAPED_UNICODE);
            } else{
                $log_data['ten_min'] = $ten;
                $content.= "> 超过{$idle_state_notice_limit}个请查看阿里云日志 RequestId： ".$request_id.PHP_EOL;
            }
            $need_send = true;
        } else {
            $content.="> 暂无数据".PHP_EOL;
        }

        if (!empty($log_data)) {
            Logbw8::getInstance()->write_log('error',$request_id,[
                'title' => '呼叫中心客服状态异常预警',
                'script' => __FILE__,
                'data' => $log_data,
            ]);
        }
        if ($need_send) {
            $report = ['title' => $title, 'text' => $content];
            $this->sendReport($report);
        }
    }

    protected $noticeUri = 'https://oapi.dingtalk.com/robot/send?access_token=21f901a7edc9d41ad9c056205c837e6ab3356840c21ace3f6755f0d57424b2d2';

    protected function sendReport($report)
    {
        $robot = new DingDingRobot($this->noticeUri);
        $robot->setMarkdownType()->atAll()->setContent($report)->send();
    }
}

class ServiceIdleStateCheck extends sample_system
{
    private $actionPage;

    public function __construct($args)
    {
        parent::__construct($args);
        $this->actionPage = new ServiceIdleStateCheck_page($this->dbo, $this->page);
    }

    public static function main($args)
    {
        $myPg = new ServiceIdleStateCheck($args);
        $myPg->actionPage->process();
    }
}

ServiceIdleStateCheck::main(array(
    'mysql6'
));

```
