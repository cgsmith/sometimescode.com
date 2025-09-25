+++
date = '2015-07-26T00:00:00Z'
draft = false
title = 'Using Silex to Refactor a Legacy PHP Application'
categories = ['programming']
tags = ['php', 'silex', 'refactoring', 'legacy-code', 'modernization']
+++

We have a legacy application here where I work, MRS. In my last post I talked about putting in place a front controller as the next logical step. I started to take a different, more thoughtful, approach after discussing with my co-workers. I used Silex.

## Hello Silex!

I am not going to introduce you to Silex, their website has all the info you need. I was trying to wrap my head around writing a front controller to handle all the old routes. Once that is done, then my plan was to start refactoring portions of the code base.

The CEO hinted at just rewriting the entire application. Instead of doing that we discovered that instead of investing all the time up front to write old routes we should just rewrite portions of code every day. Here is a quick real-world example: we have transfer scripts that run via cron daily. The scripts are pretty simple stuff and go as follows:

1. Gather customer orders ready for transfer
2. ZIP documents up
3. SFTP, SCP, etc. to the customer location
4. Possibly trigger an email

Pretty simple, right? Well the application mutated over almost two decades. The customers and scripts changed and were copies of copies. Eventually, like DNA, the scripts become a vacuum for old useless information.

## I refactored it

Here is what the transfer script looked like before the rewrite.

**What's wrong with this #php?**

```php
<?php
if ( count($apsBatchResults) > 0 ) {
    foreach ( $apsBatchResults AS $orderID=>$resultsSet ) {
        extract($resultsSet);
        $imageFilePathInfo = pathinfo($imagePath);
        $manifestFilePathInfo = pathinfo($manifestPath);
        $encryptedImagePath = "$imagePath.gpg";
        $encryptedManifestPath = "$manifestPath.gpg";
/*
        $GLOBALS['logger']->log("$orderID : Encrypting $imagePath");
        rms_encryptFile ($imagePath, $encryptedImagePath);
        $GLOBALS['logger']->log("$orderID : Encrypting $manifestPath");
        rms_encryptFile ($manifestPath, $encryptedManifestPath);
*/
        /**
         *	2013-10-09:	Shaheeb R.
         *	If the file is >=101 pages, then these should be sent to customer and not GWL.
         **/
        if ( $pageCount >= 101 ) {
            $GLOBALS['logger']->log("$orderID : PageCount: $pageCount >= 101.  Sending to customer");
            if ( is_file($imagePath) ) {
                $GLOBALS['logger']->log("$orderID : Transferring the documents to $transferDest");
                $transferStatus = TransferHandler::secftpUWPackage($imagePath);
                if ( $transferStatus === true ) {
                    $noteComment = "Transfer of $imagePath to Synodex due to pagecount: $pageCount successful";
                }else {
                    $noteComment = "Transfer of $imagePath to Synodex due to pagecount: $pageCount failed";
                }
                $GLOBALS['logger']->log("$orderID: $noteComment");
                create_note($noteComment, "YES", 6, $orderID);
                /**
                 *	Email on Synodex Transfer
                 *	Removed email from recipient list on 12/18/2013 per request in SW Ticket#8863
                **/
                $emailLogParms = array(
                    "recipientAddress"=>"email@email.com",
                    "emailSubject"=>"MRS Transfer to customer on MRS Case#: $orderID",
                    "senderName"=>"**MRS Script**",
                    "emailBody"=>$noteComment
                );
                GWLTransferHandler::phpEmail($emailLogParms);
            }else {
                $GLOBALS['logger']->log("$orderID : Problem encrypting $encryptedManifestPath and/or $encryptedImagePath");
            }
        }else {
            if ( is_file($imagePath) && is_file($manifestPath) ) {
                $GLOBALS['logger']->log("$orderID : Transferring the documents to $transferDest");
                system ("syncPayloads $manifestPath $transferDest");
                system ("syncPayloads $imagePath $transferDest");
            }else {
                $GLOBALS['logger']->log("$orderID : Problem encrypting $encryptedManifestPath and/or $encryptedImagePath");
            }
        }
    }
}
```

It is pretty bad, but we can make it better. I found all the things in common as I numbered out above and rewrote the transfer script. Programmer's note: when rewriting, you have to find common ground – note my require_once and dependency on Net_SFTP – yes it is bad, but look where it came from!

```php
<?php
class TransferHandler
{
    /**
     * @var string ftp host
     */
    protected $targetFTPHost = "sftp.customer.com";

    /**
     * @var string ftp user
     */
    protected $targetFTPUser = "user1";

    /**
     * @var string ftp password
     */
    protected $targetFTPPass = 'hi';

    /**
     * @var string ftp destination folder
     */
    protected $targetFTPDestinationFolder = 'to-synx';

    /**
     * @var string where to rsync the files
     */
    protected $transferDestination = '/';

    /**
     * @var string imagePath
     */
    protected $imagePath = null;

    /**
     * @var string manifestPath
     */
    protected $manifestPath = null;

    /**
     * @var bool useSFTP determines if files should transmit via SFTP
     */
    protected $useSFTP = null;

    /**
     * @var string list of comma separated emails to send confirmation to
     */
    public $emailAddress = 'email@mrs.com';

    /**
     * @var string email from name
     */
    protected $emailFromName = '**MRS Script**';

    /**
     * @var string from email that is sent out
     */
    protected $emailFrom = 'it@mrs.com';

    /**
     * @const METHOD_SFTP 100 used in setMethod()
     */
    const METHOD_SFTP = 100;

    public function sendFiles()
    {
        return ($this->useSFTP) ? $this->secftpUWPackage() : $this->syncPayloads();
    }

    /**
     * Uses bin/syncPayloads, which just aliases rsync, to sync the files to destination
     * @todo Use a better method that can verify successful sync
     */
    protected function syncPayloads()
    {
        system("syncPayloads $this->manifestPath $this->transferDestination");
        system("syncPayloads $this->imagePath $this->transferDestination");
        return true;
    }

    /**
     * This function leverages built in PHP support for SSH to SCP the document payload to the remote server.
     * @return bool
     * @throws Exception
     * @todo remove the Net_SFTP dependency ^_^ - cgsmith
     */
    protected static function secftpUWPackage()
    {
        define('NET_SFTP_LOGGING', NET_SFTP_LOG_COMPLEX);
        $sftpConnection = new Net_SFTP($this->targetFTPHost);
        if (!$sftpConnection->login($this->targetFTPUser, $this->targetFTPPass)) {
            throw new Exception('SFTP Login Failed on transferGreatWestLife.php');
        }
        return $sftpConnection->put(
            $this->targetFTPDestinationFolder . '/' . strtolower(basename($this->imagePath)),
            $this->imagePath,
            NET_SFTP_LOCAL_FILE);
    }

    /**
     * @param $subject
     * @param $body
     * @return bool
     */
    public static function phpEmail($subject, $body)
    {
        require_once($_SERVER['DOCUMENT_ROOT'] . "/lib/class.phpmailer.php");
        $mail = new PHPMailer();
        $mail->AddAddress($this->emailAddresses);
        $mail->FromName($this->emailFromName);
        $mail->From = $this->emailFrom;
        $mail->Sender = $this->emailFrom;
        $mail->AddReplyTo($this->emailFrom);
        $mail->Subject = $subject;
        $mail->Body = $body;
        $mail->WordWrap = 200;
        return $mail->Send();
    }

    /**
     * Verifies files exists and sets protected variables for other methods
     *
     * @param $imagePath
     * @param $manifestPath
     * @return bool
     * @throws Exception
     */
    public function verifyFiles($imagePath, $manifestPath)
    {
        if (is_null($this->useSFTP)) {
            throw new Exception('setMethod() must be called first to determined verifyFiles()');
        }
        if ($this->useSFTP) {
            $this->imagePath = $imagePath;
            return is_file($imagePath);
        } else {
            $this->imagePath = $imagePath;
            $this->manifestPath = $manifestPath;
            return is_file($imagePath) && is_file($manifestPath);
        }
    }

    /**
     * Tells other methods to utilize SFTP if count is higher than constant
     *
     * @param $count
     */
    public function setMethod($count)
    {
        $this->useSFTP = ($count > self::METHOD_SFTP) ? true : false;
    }

    /**
     * @return bool
     */
    public function getMethod()
    {
        return $this->useSFTP;
    }
}
```

## How to start including Silex

As I said before, I struggled with including a front controller because the application is just so tangled. I started by taking something that is a smaller subset of the application like transfer scripts and broke them off. The refactoring process goes a little like this:

1. Setup your Silex application next to your legacy app.
2. Choose one part of a process (creating customers, transferring orders, database services, etc.)
3. Move that process into your Silex application and build your routes. My route ended up looking like: `localhost/customer/{id}/transferOrders`
4. Reconnect your application and test.

The fourth step is most likely the most important and easiest to gloss over. Remember how the lack of tests come back to haunt you? Yes, you will need to start writing unit tests now to prevent regression. You'll notice this being much easier now with Silex by our side!

## My Favorite Silex Pairings

Silex runs well by itself, but I am starting to find some favorite pairings of mine.

- **Swiftmailer** - Used to assist with SMTP mailing throughout the application
- **Monolog** - An abstraction for logging. Pass through a handler and you have an API that is the same throughout your application for logging events
- **Config Service Provider** - Allows you to use php, json, yaml, and toml for you configuration files
- **Flysystem** - Abstraction of file system interactions – this was useful when customers would transfer orders differently. Some where SFTP while others were FTP. I used Flysystem as an abstraction to this process.
- **Pimple Dumper** - This is a useful script to include in `require-dev` if you have PHP Storm. It allows auto-completion on the Silex dependency injection container. Sweet!

## In Conclusion

I cannot tell you what you need to do in your situation. In fact, it is not necessarily beneficial to show you step-by-step what I have done… your mileage may vary.

I badly want to rewrite this code. I just want to say, "Let's start from square one". Heck, I have said it! I have spoke too soon though and should not have. It can be refactored and now my thought is that it should for the businesses sake. If we dedicate a lot of time rewriting a whole new application we will find ourselves in similar pitfalls in months to come.