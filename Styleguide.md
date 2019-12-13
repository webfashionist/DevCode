## Styleguide

### Foreword

>Indeed, the ratio of time spent reading [code] versus writing is well over 10 to 1. We are constantly reading old code as part of the effort to write new code. Because this ratio is so high, we want the reading of code to be easy, even if it makes the writing harder. Of course there’s no way to write code without reading it, so making it easy to read actually makes it easier to write.    
**Robert C. Martin**

To be able to code efficiently and productively, it is essential that the code we want to expend or modify is clean and easy to read.        
This styleguide aims to set ground rules that should help us to reach that goal and write code with a consistent structure.     
When we all agree on specific rules about how our code has to look like, we're taking a big step in making our code more readable.

Keep in mind that this guide is not meant to contain the general principles about clean code.    
There have been written many guides about that before and I will link one of them for a more general understanding about what it means to code well and clean.    
This styleguide rather has the intent to, as I said before, set ground rules and pick up some of the most important code principles.     

To first get a general understanding I would like to ask you to read this guide:   
https://github.com/jupeter/clean-code-php

### Basics

- **camelCase** for naming in code (method names, variables, etc.), 
- **snake_case** for all database things (table names, column names)
- New line after class and method declaration (except for loops and conditionals)
    ```php
    class Example
    {
        public function example()
        {
            if($expression) {
                ...
            }
            while($expression) {
                ...
            }
        }
    }
    ```
- Data-Types of parameters and return values should be declared (if possible)
    ```php
    public function add(int $a, int $b): int 
	{
		...
	}
    ```
- Spaces between expressions
To make expressions more readable it should look like this: `$a = 10` instead of  `$a=10`
- Every method should have a PHPDoc block
    ```php
    /**
     * @param int $number
     * @return bool
     */
    public function example(int $number): bool 
	{
		...
	}
    ```


### No more than three levels of indentation

If you have more than three levels of indentation then it is very likely that your function / method does too much.    
Therefore it is more readable if you split the code in small, neat functions.

Consider for example the following method: 

```php
public function sendMails()
{
    $mail_count = $this->checkMailCount();
    if ($mail_count == null || $mail_count < 400) {
        $con = $this->setupConnection(MDATABASE, MASTER_USER_RW, MPASSWORD_RW, HOST);
        $query = "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC";
        $delete_query = "DELETE FROM `mail_cron` WHERE id = :id";
        try {
            $sql = $con->prepare($query);
            $sql->execute();
            $mails = $sql->fetchAll(PDO::FETCH_OBJ);
            if ($mails && is_array($mails) && count($mails) > 0) {
                // emails available for sending
                foreach ($mails as $mail) {
                    if ($mail_count == null || $mail_count < 400) {
                        $subject = htmlspecialchars_decode($mail->subject, ENT_QUOTES);
                        $message = htmlspecialchars_decode(html_entity_decode($mail->message), ENT_QUOTES);
                        Mail::send($mail->recipient, $subject, $message, $mail->header);
                        if ($mail->type && substr($mail->type, 0, strlen("reminder_")) == "reminder_") {
                            // update reminder count ?
                            // $this->incrementReminderCount(substr($mail->type,strlen("reminder_")));
                        }
                        $sql2 = $con->prepare($delete_query);
                        $sql2->bindParam(":id", $mail->id, PDO::PARAM_INT);
                        $sql2->execute();
                        $mail_count++;
                    }
                }
                $this->putMailCount($mail_count);
            }
            return true;
        } catch (PDOException $ex) {
            trigger_error("Could not fetch any emails from mail_cron. " . $ex->getMessage(), E_USER_WARNING);
            return false;
        }
    } else {
        // mail count limit already reached - do not send any other emails now
        return false;
    }
}
```

Because the method is doing too much (fetching all mails from the database, looping through all, formatting the mail and finally send it), the code is hard to read and therefore hard to modify.      
Also if you just want to modify the part which sends the mail, you first have to mentally focus on the part which comes before (fetching all emails) but does not really have much to do with how the mail is being sent.

[How this can be refactored](RefactoringSendMails.md)

### Try to use as few parameters as possible

Again, if a function / method has too many parameters, it does too much.   
The perfect amount of parameters is zero.     
One to three parameters are acceptable and anything more than that should be avoided / refactored.    

If your method has more than three parameters and you need all of them,    
then it is in most cases better to make a class out of it and split the method up in smaller parts.


### Use Collection instead of loops

Many of the use-cases of loops can be simplified with the Collection class.    
You can convert an array to a Collection by using `collect($array)` or `Collection::fromArray($array)`.

- **Example 1 (Value manipulation)**
    ```php
    public function example(): array
    {
        $numbers = range(0, 100);
        foreach($numbers as $key => $item) {
            $numbers[$key] = someFunction($item);
        }
        return $numbers;
    }
    ```
    With Collection: 
    ```php
    public function example(): array
    {
        return collect(range(0, 100))->each(function($number) {
           return someFunction($number); 
        });
    }
    ```
- **Example 2 (Filtering)**
    ```php
    public function example(): array
    {
        $numbers = range(0, 100);
        $evenNumbers = [];
        foreach($numbers as $number) {
            if($number % 2 === 0) {
                $evenNumbers[] = $number;
            }
        }
        return $evenNumbers;
    }
    ```
    With Collection: 
    ```php
    public function example(): array
    {
        return collect(range(0, 100))->filter(function($number) {
           return ($number % 2 === 0);
        })->all();
    }
    ```
- **Example 3 (Chaining)**
    ```php
    public function example(): array
    {
        return collect(range(0,100))->filter(function($number) {
            return ($number % 2 === 0);
        })->each(function($number) {
            return $number * 10;
        });
    }
    ```
    
### Links

Clean Code: https://github.com/jupeter/clean-code-php    
Design Patterns: https://designpatternsphp.readthedocs.io/en/latest/