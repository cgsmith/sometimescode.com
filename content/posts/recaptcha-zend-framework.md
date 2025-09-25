+++
date = '2015-03-31T00:00:00Z'
draft = false
title = 'How to Integrate reCAPTCHA with Zend Framework 1.12'
categories = ['programming']
tags = ['php', 'zend-framework', 'recaptcha', 'google', 'forms', 'validation']
+++

## I'm not a robot – Google's reCAPTCHA 2.0

Google's new reCAPTCHA has been released to the wild. At LPi I had to integrate the customer's forms to allow for this option for integration. We use Zend Framework 1.12 for our applications. ZF1 comes with an integration for recaptcha 1.0 but nothing for recaptcha 2.0.

It was a bit of a challenge to get the captcha working, but I think it will leave a lasting impression for those with Zend Framework 1.12 versions. It should be written in a well documented manner and in a way that is easily changeable if Google decides to change their API or terminate a service… but they would never do that.

Enough joking aside, let me show you how I programmed it!

## Creating a new ZF1 form element, validator, and view helper

I should mention that this is all on GitHub and available through Packagist as well. The first thing I did was extend Zend_Form_Element and overload the constructor to make sure that the options were set properly. Note that I also set the validator up to point to the custom validator here.

```php
<?php
namespace Cgsmith\Form\Element;

/**
 * Class Recaptcha
 * Renders a div for google recaptcha to allow for use of the Google Recaptcha API
 *
 * @package Cgsmith
 * @license MIT
 * @author  Chris Smith
 */
class Recaptcha extends Zend_Form_Element
{
    /** @var string specify formRecaptcha helper */
    public $helper = 'formRecaptcha';

    /** @var string siteKey for Google Recaptcha */
    protected $_siteKey = '';

    /** @var string secretKey for Google Recaptcha */
    protected $_secretKey = '';

    /**
     * Constructor for element and adds validator
     *
     * @param array|string|Zend_Config $spec
     * @param null $options
     * @throws Zend_Exception
     * @throws Zend_Form_Exception
     */
    public function __construct($spec, $options = null)
    {
        if (empty($options['siteKey']) || empty($options['secretKey'])) {
            throw new Zend_Exception('Site key and secret key must be specified.');
        }
        $this->_siteKey = trim($options['siteKey']); // trim the white space if there is any just to be sure
        $this->_secretKey = trim($options['secretKey']); // trim the white space if there is any just to be sure
        $this->addValidator('Recaptcha', false, ['secretKey' => $this->_secretKey]);
        $this->setAllowEmpty(false);
        parent::__construct($spec, $options);
    }
}
```

Once the element is called it looks for the helper (formRecaptcha) and spits out the appropriate div for Google's API to interpret properly. When the form validates it will run `\Cgsmith\Validate\Recaptcha`.

```php
<?php
namespace Cgsmith\Validate;

/**
 * Class ValidateRecaptcha
 * Handle validation against Google API
 *
 * @package Cgsmith
 * @license MIT
 * @author Chris Smith
 * @link   https://github.com/google/recaptcha
 */
class ValidateRecaptcha extends Zend_Validate_Abstract
{
    /** @var string secret key */
    protected $_secretKey;

    /** @const string invalid captcha */
    const INVALID_CAPTCHA = 'invalidCaptcha';

    /** @const string invalid captcha */
    const CAPTCHA_EMPTY = 'captchaEmpty';

    /** @const string URL to where requests are posted */
    const SITE_VERIFY_URL = 'https://www.google.com/recaptcha/api/siteverify';

    /** @const string http method for communicating with google */
    const POST_METHOD = 'POST';

    /** @const string peer key for communication */
    const PEER_KEY = 'www.google.com';

    protected $_messageTemplates = array(
        self::INVALID_CAPTCHA => 'The captcha was invalid',
        self::CAPTCHA_EMPTY   => 'The captcha must be completed'
    );

    /**
     * @param $options
     */
    public function __construct($options) {
        $this->_secretKey = $options['secretKey'];
    }

    /**
     * Validate our form's element
     *
     * @param mixed $value
     * @param null $context
     * @return bool
     */
    public function isValid($value, $context = null)
    {
        if (empty($value)) {
            $this->_error(self::CAPTCHA_EMPTY);
            return false;
        }

        if (!$this->_verify()) {
            $this->_error(self::INVALID_CAPTCHA);
            return false;
        }

        return true;
    }

    /**
     * Calls the reCAPTCHA siteverify API to verify whether the user passes the captcha test.
     *
     * @return boolean
     * @link   https://github.com/google/recaptcha
     */
    protected function _verify()
    {
        $queryString = http_build_query([
            'secret'   => $this->_secretKey,
            'response' => $this->_value,
            'remoteIp' => $_SERVER['REMOTE_ADDR']
        ]);

        /**
         * PHP 5.6.0 changed the way you specify the peer name for SSL context options.
         * Using "CN_name" will still work, but it will raise deprecated errors.
         */
        $peerKey = version_compare(PHP_VERSION, '5.6.0', '<') ? 'CN_name' : 'peer_name';
        $context = stream_context_create([
            'http'  => [
                'header'      => "Content-type: application/x-www-form-urlencoded\r\n",
                'method'      => self::POST_METHOD,
                'content'     => $queryString,
                'verify_peer' => true,
                $peerKey      => self::PEER_KEY
            ]
        ]);
        $jsonObject = json_decode(file_get_contents(self::SITE_VERIFY_URL,false,$context));

        return $jsonObject->success;
    }
}
```

The validation will reach out to Google's API to determine if the user is valid. The cool thing about the new API: you get statistics and it will ask the user to type a captcha if it is not 100% sure they are not a bot.

## Requirements and Notes on Implementing

I will keep the readme updated on the GitHub repository but here is a quick bullet list of what you will need to do to use Google reCAPTCHA:

- Be sure to set your helper paths and prefix paths in your application
- You will need to get the site key and api key from Google's reCAPTCHA admin interface

*This code is available on [GitHub](https://github.com/cgsmith/zf1-recaptcha-2) and through Packagist.*