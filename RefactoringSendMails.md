Let's take a rather complex and not-so-clean function and try to refactor it together:

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

First of all, we can reduce the indentation by changing the first conditional like this:

```php
if ($mail_count == null || $mail_count < 400) {
    ...
} else {
    return false;   
}
``` 

```php
if($mail_count >= 400) {
    return false;
}
...
```

Now it looks like this:

```php
public function sendMails()
{
    $mail_count = $this->checkMailCount();
    if($mail_count >= 400) {
        return false;
    }
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
}
```

Next we can extract the fetching of the mails in its own method: 

```php
public function fetchMailCronMails(): array
{
    $con = $this->setupConnection(MDATABASE, MASTER_USER_RW, MPASSWORD_RW, HOST);
    $query = "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC";
    try {
        $sql = $con->prepare($query);
        $sql->execute();
        $mails = $sql->fetchAll(PDO::FETCH_OBJ);
    } catch (PDOException $ex) {
        trigger_error("Could not fetch any emails from mail_cron. " . $ex->getMessage(), E_USER_WARNING);
    }
    return $mails ?? [];
}
```

With the new database class, we can even simplify this further:

```php
public function fetchMailCronMails(): array
{
    $Query = Query::evaleaDB(
        "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC"
    );
    $Query->setErrorMessage("Could not fetch any emails from mail_cron.");
    return $Query->make()->all();
}
```

Good! Now the code looks like this:

```php
public function sendMails()
{
    $mail_count = $this->checkMailCount();
    if($mail_count >= 400) {
        return false;
    }
    $delete_query = "DELETE FROM `mail_cron` WHERE id = :id";
    $mails = $this->fetchMailCronMails();
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
}

public function fetchMailCronMails(): array
{
    $Query = Query::evaleaDB(
        "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC"
    );
    $Query->setErrorMessage("Could not fetch any emails from mail_cron.");
    return $Query->make()->all();
}
```

So, now maybe we add an own method for sending all mails:

```php
public function sendMailCronMails(array $mails)
{
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
}
```

But now, we are missing two variables. `$delete_query` and `$mail_count`.    
We can fully extract `$delete_query` from the main method, because it isn't used there anymore.    
But `$mail_count` is still a problem. We can either pass it as a parameter or call the `checkMailCount()` function in this method again.    
Let's try to use as few parameters as possible and solve it by calling `checkMailCount()`.    
It is now called twice, which is not ideal as it fetches something from the database, but maybe we can figure something out later.   

It should now look like this:

```php
public function sendMails()
{
    $mail_count = $this->checkMailCount();
    if($mail_count >= 400) {
        return false;
    }
    $mails = $this->fetchMailCronMails();
    $this->sendMailCronMails($mails);
    return true;
}

public function sendMailCronMails(array $mails)
{
    $delete_query = "DELETE FROM `mail_cron` WHERE id = :id";
    $mail_count = $this->checkMailCount();
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
}

public function fetchMailCronMails(): array
{
    $Query = Query::evaleaDB(
        "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC"
    );
    $Query->setErrorMessage("Could not fetch any emails from mail_cron.");
    return $Query->make()->all();
}
```

The `sendMailCronMails()` method seems to do a bit too much.    
For each mail it    
**a)** formats the subject and message of the mail    
**b)** sends the mail   
**c)** checks if the mail is a reminder mail and does ... nothing(?)   
**d)** deletes the entry for that mail in the mail_cron table    

Let's see if we can simplify that.    
First we create a method for sending the mail:

```php
public function sendMailCronMail($mail)
{
    $subject = htmlspecialchars_decode($mail->subject, ENT_QUOTES);
    $message = htmlspecialchars_decode(html_entity_decode($mail->message), ENT_QUOTES);
    Mail::send($mail->recipient, $subject, $message, $mail->header);
}
```

Then we can create a method for removing the mail entry from the database:

```php
public function deleteMailCronEntry(int $id): bool
{
    return Query::evaleaDB(
        "DELETE FROM `mail_cron` WHERE id = :id",
        [":id" => [$id, \PDO::PARAM_INT]]
    )->exec();
}
```

As the special case for reminder mails is currently not used, we can remove it for now.

And now it should look like this:

```php
public function sendMails()
{
    $mail_count = $this->checkMailCount();
    if($mail_count >= 400) {
        return false;
    }
    $mails = $this->fetchMailCronMails();
    $this->sendMailCronMails($mails);
    return true;
}

public function sendMailCronMails(array $mails)
{
    $mail_count = $this->checkMailCount();
    if ($mails && is_array($mails) && count($mails) > 0) {
        // emails available for sending
        foreach ($mails as $mail) {
            if ($mail_count == null || $mail_count < 400) {
                $this->sendMailCronMail($mail);
                $this->deleteMailCronEntry($mail->id);
                $mail_count++;
            }
        }
        $this->putMailCount($mail_count);
    }
}

public function deleteMailCronEntry(int $id): bool
{
    return Query::evaleaDB(
        "DELETE FROM `mail_cron` WHERE id = :id",
        [":id" => [$id, \PDO::PARAM_INT]]
    )->exec();
}

public function sendMailCronMail($mail)
{
    $subject = htmlspecialchars_decode($mail->subject, ENT_QUOTES);
    $message = htmlspecialchars_decode(html_entity_decode($mail->message), ENT_QUOTES);
    Mail::send($mail->recipient, $subject, $message, $mail->header);
}

public function fetchMailCronMails(): array
{
    $Query = Query::evaleaDB(
        "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC"
    );
    $Query->setErrorMessage("Could not fetch any emails from mail_cron.");
    return $Query->make()->all();
}
```

We can also make the `sendMailCronMails()` method a bit more clean by changing the conditionals:

```php
public function sendMailCronMails(array $mails)
{
    if(empty($mails)) {
        return;
    }

    $mail_count = $this->checkMailCount();
    // emails available for sending
    foreach ($mails as $mail) {
        if($mail_count >= 400) {
            continue;
        }
        $this->sendMailCronMail($mail);
        $this->deleteMailCronEntry($mail->id);
        $mail_count++;
    }
    $this->putMailCount($mail_count);
}
```

Looks pretty good!
We could now now clean the `sendMails()` method a bit by inlining the variables:

```php
public function sendMails()
{
    if($this->checkMailCount() >= 400) {
        return false;
    }
    $this->sendMailCronMails($this->fetchMailCronMails());
    return true;
}
```

Now, the code looks quite clean and readable to me. Nothing obvious that could be changed further.      
But wait, we still have the issue with the `checkMailCount()` method that is called twice        
and also we have now many small methods in a class that isn't really related to those methods.    

Let's make it into a separate class:

```php
class SendMailsCronjob
{
    public function sendMails()
    {
        if($this->checkMailCount() >= 400) {
            return false;
        }
        $this->sendMailCronMails($this->fetchMailCronMails());
        return true;
    }

    public function sendMailCronMails(array $mails)
    {
        if(empty($mails)) {
            return;
        }

        $mail_count = $this->checkMailCount();
        // emails available for sending
        foreach ($mails as $mail) {
            if($mail_count >= 400) {
                continue;
            }
            $this->sendMailCronMail($mail);
            $this->deleteMailCronEntry($mail->id);
            $mail_count++;
        }
        $this->putMailCount($mail_count);
    }

    public function deleteMailCronEntry(int $id): bool
    {
        return Query::evaleaDB(
            "DELETE FROM `mail_cron` WHERE id = :id",
            [":id" => [$id, \PDO::PARAM_INT]]
        )->exec();
    }

    public function sendMailCronMail($mail)
    {
        $subject = htmlspecialchars_decode($mail->subject, ENT_QUOTES);
        $message = htmlspecialchars_decode(html_entity_decode($mail->message), ENT_QUOTES);
        Mail::send($mail->recipient, $subject, $message, $mail->header);
    }

    public function fetchMailCronMails(): array
    {
        $Query = Query::evaleaDB(
            "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC"
        );
        $Query->setErrorMessage("Could not fetch any emails from mail_cron.");
        return $Query->make()->all();
    }
}
```

But we now have two methods `putMailCount()` and `checkMailCount()` which are belonging to the other class.      
We can now either pass an instance of that other class in our constructor and call the methods on that instance,      
or we just accept a mail count in the constructor and then increment it internally and add a getter for that variable.   

Let's do the latter:

```php
class SendMailsCronjob
{
    private $mailCount = 0;

    public function __construct(int $mailCount)
    {
        $this->mailCount = $mailCount;
    }

    public function sendMails(): bool
    {
        if($this->mailCount >= 400) {
            return false;
        }
        $this->sendMailCronMails($this->fetchMailCronMails());
        return true;
    }

    public function getMailCount(): int
    {
        return $this->mailCount;
    }

    public function sendMailCronMails(array $mails)
    {
        if(empty($mails)) {
            return;
        }

        foreach ($mails as $mail) {
            if($this->mailCount >= 400) {
                continue;
            }
            $this->sendMailCronMail($mail);
            $this->deleteMailCronEntry($mail->id);
            $this->mailCount++;
        }
    }

    public function deleteMailCronEntry(int $id): bool
    {
        return Query::evaleaDB(
            "DELETE FROM `mail_cron` WHERE id = :id",
            [":id" => [$id, \PDO::PARAM_INT]]
        )->exec();
    }

    public function sendMailCronMail($mail)
    {
        $subject = htmlspecialchars_decode($mail->subject, ENT_QUOTES);
        $message = htmlspecialchars_decode(html_entity_decode($mail->message), ENT_QUOTES);
        Mail::send($mail->recipient, $subject, $message, $mail->header);
    }

    public function fetchMailCronMails(): array
    {
        $Query = Query::evaleaDB(
            "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC"
        );
        $Query->setErrorMessage("Could not fetch any emails from mail_cron.");
        return $Query->make()->all();
    }
}
```

Cool! There are still a few small imperfections:
**a)** `sendMails()` and `sendMailCronMails()` are a bit confusing. 
Now that we have a own class we can shorten the method names.
**b)** We no longer need the conditional in `sendMailCronMails()` because if `$mails` is empty, it just skips the loop.
**c)** The methods that aren't used from the outside can be made private.

Finally it should look like this:

```php
class SendMailsCronjob
{
    private $mailCount = 0;

    public function __construct(int $mailCount)
    {
        $this->mailCount = $mailCount;
    }

    public function run(): bool
    {
        if($this->mailCount >= 400) {
            return false;
        }
        $this->sendMails($this->fetchMails());
        return true;
    }

    public function getMailCount(): int
    {
        return $this->mailCount;
    }

    private function sendMails(array $mails)
    {
        foreach ($mails as $mail) {
            if($this->mailCount >= 400) {
                continue;
            }
            $this->sendMail($mail);
            $this->deleteEntry($mail->id);
            $this->mailCount++;
        }
    }

    private function sendMail($mail)
    {
        $subject = htmlspecialchars_decode($mail->subject, ENT_QUOTES);
        $message = htmlspecialchars_decode(html_entity_decode($mail->message), ENT_QUOTES);
        Mail::send($mail->recipient, $subject, $message, $mail->header);
    }

    private function deleteEntry(int $id): bool
    {
        return Query::evaleaDB(
            "DELETE FROM `mail_cron` WHERE id = :id",
            [":id" => [$id, \PDO::PARAM_INT]]
        )->exec();
    }

    private function fetchMails(): array
    {
        $Query = Query::evaleaDB(
            "SELECT * FROM `mail_cron` WHERE timestamp < UNIX_TIMESTAMP(NOW()) ORDER BY timestamp ASC"
        );
        $Query->setErrorMessage("Could not fetch any emails from mail_cron.");
        return $Query->make()->all();
    }
}
```

And the method in the other class:

```php
public function sendMails()
{
    $SendMailsCronjob = new SendMailsCronjob($this->checkMailCount());
    $SendMailsCronjob->run();
    $this->putMailCount($SendMailsCronjob->getMailCount());
}
```

Now, it may not be 100% perfect, but I would say it is way more readable than the original.
And also by having several small methods it is easier to modify and extend.

Note: To save space and therefore have a better overview, the PHPDoc blocks were omitted in this tutorial.
