+++
title = 'Creating a simple, extensible CodeIgniter authentication library'
date = 2010-05-10T15:00:00-05:00
draft = false
tags = ['programming', 'php', 'codeigniter']
+++
While it is true that there are a whole [slew of third party CodeIgniter authentication systems](http://codeigniter.com/wiki/Category:Contributions::Libraries::Authentication/), there are a few compelling reasons to write your own. In my own experience, third party authentication systems are too feature-rich and complex for a particular application or they are not currently being supported, which means they may not work with newer versions of CI or may have outstanding functional and security bugs. Instead of tackling a new codebase to fight these issues, you may be better off rolling your own solution. 

This tutorial is for those of you who may need some help starting out.

<!--more-->

# Features

The feature list of the library is sparse to keep it simple. A user may register, log in, and log out. A user has a level that decides their role. For example, a level 1 user may be a 'member' and a level 10 user may be a 'super administrator'. However, understand that you are not defining _named_ roles, only an integer. Anyway, you may restrict what pages a user can access using that level. Lastly, this library will keep track of the day the user created their account and the last time they logged in.

For now, lets get to the implementation.

## Implementation

First, you'll need to create a mysql database (I assume you know how to do this, but if you don't leave a comment and I'll explain.) Then, you should import this mysql table into your database either using this file or doing it by hand. You'll notice that I am storing the passwords as Sha-1 hashes.

    CREATE TABLE IF NOT EXISTS `users` (
      `ID` int(11) NOT NULL AUTO_INCREMENT,
      `email` varchar(50) NOT NULL,
      `password` varchar(40) NOT NULL,
      `level` int(11) NOT NULL DEFAULT '1',
      `creation_date` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
      `last_login` timestamp NULL DEFAULT NULL,
      PRIMARY KEY (`ID`)
    ) ENGINE=MyISAM  DEFAULT CHARSET=latin1 AUTO_INCREMENT=3 ;

Next is the implementation of the library. You'll place this in the libraries folder of your application folder.

    <?php if (!defined('BASEPATH'))
    exit('No direct script access allowed');

    class Authex
    {

        function Authex()
        {
            $CI =& get_instance();

            //load libraries
            $CI->load->database();
            $CI->load->library("session");
        }

        function get_userdata()
        {
            $CI =& get_instance();

            if (!$this->logged_in()) {
                return false;
            } else {
                $query = $CI->db->get_where("users", array("ID" => $CI->session->userdata("user_id")));
                return $query->row();
            }
        }

        function logged_in()
        {
            $CI =& get_instance();
            return ($CI->session->userdata("user_id")) ? true : false;
        }

        function login($email, $password)
        {
            $CI =& get_instance();

            $data = array(
                "email" => $email,
                "password" => sha1($password)
            );

            $query = $CI->db->get_where("users", $data);

            if ($query->num_rows() !== 1) {
                /* their username and password combination
                * were not found in the databse */

                return false;
            } else {
                //update the last login time
                $last_login = date("Y-m-d H-i-s");

                $data = array(
                    "last_login" => $last_login
                );

                $CI->db->update("users", $data);

                //store user id in the session
                $CI->session->set_userdata("user_id", $query->row()->ID);

                return true;
            }
        }

        function logout()
        {
            $CI =& get_instance();
            $CI->session->unset_userdata("user_id");
        }

        function register($email, $password)
        {
            $CI =& get_instance();

            //ensure the email is unique
            if ($this->can_register($email)) {
                $data = array(
                    "email" => $email,
                    "password" => sha1($password)
                );

                $CI->db->insert("users", $data);

                return true;
            }

            return false;
        }

        function can_register($email)
        {
            $CI =& get_instance();

            $query = $CI->db->get_where("users", array("email" => $email));

            return ($query->num_rows() < 1) ? true : false;
        }
    }

## How to use

```
class Hello_world extends Controller
{
    /* Only a logged in user can access anything
        in this controller */
    function __construct()
    {
        parent::Controller();

        //load the library
        $this->load->library("authex");

        //this is how we protect it
        if( ! $this->authex->logged_in())
        {
            //they are not logged in
            redirect("user/login");  //for example
        }

        //continue with the rest of the constructor
    }
}
```
```
class Admin_area extends Controller
{
    /* Only a logged in user with level 5 or better (an admin)
        can access anything in this controller */
    function __construct()
    {
        parent::Controller();

        //load the library
        $this->load->library("authex");

        //this is how we protect it
        if(! $this->authex->logged_in())
        {
            redirect("user/login");
        }
        else
        {
            //get the user info
            $user_info = $this->authex->get_userdata();

            //make sure they are level 5 or better
            if($user_info->level < 5)
            {
                redirect("user/login"); //again, for example
            }

            //continue with the rest of the constructor
        }
    }
}
```

You can also protect individual functions using similar methods. Instead of putting the protection code into the controller's constructor, put the protection code into the function you wish to protect. If you have any questions about how to use the class, feel free to post a comment.

## Notes about the library

You probably noticed when I was protecting a controller by requiring a certain user level, I called the function get_userdata(). This function returns an object that allows you to access any field stored in the user table or false if the user isn't logged in. Through this function you will have access to the currently logged in users ID, email, level, creation date, and last login date (if you wanted to display them somewhere, for example.) This function is one of the avenues through which we may extend our class.

Suppose, for example, you wanted to store billing information for the user. You could add a field to the users table called shipping_id and that would be the foreign key to another table, shipping_addresses, which would store a shipping address. You would have to make two changes to the library. First, you would have to update the register() function to generate a new shipping_addresses row and insert the pk to that table / row (probably an ID field) into the users table. Second, in the get_userdata() function, you would join the users table to the shipping_addresses table by linking the shipping_id in the users table to the ID field in the shipping_addresses table and returning the object. Luckily, by using CodeIgniter's ActiveRecord class similar to how I have used it, you can modify both functions fairly trivially.

Another method you may want to add to the class is a built in method to update the users data. Were I to implement this functionality (and if there is sufficient interest, I will) would be to create a function that accepts 2 parameters: the first would be the ID number of the user to update, which you could pull straight from the session if the user were updating their own information, and an associative array as the second parameter. The associative array keys would be the field to modify, and their value would be the updated values.

If you have any questions or suggestions, please post your comments and I will try my best to answer!
