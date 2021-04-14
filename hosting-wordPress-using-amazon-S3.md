


## Task 1: Configure WordPress on Amazon EC2 

An Amazon EC2 instance containing WordPress has been automatically provisioned as part of this lab. 

In this task, you perform the initial configuration of WordPress and create a blog post. 

To the left of these instructions, copy the WordPressURL. 

This URL is a link to your WordPress website. 

Paste the URL into a new web browser tab and press Enter. 

The WordPress configuration page displays. 

Configure the following values: 

 

Site Title: Enter any title you wish 

Username:  

Password:  

Your Email:  

Search Engine Visibility: Leave deselected 

 

Choose Install WordPress. 

You are presented with a Success screen. 

Choose Log In and then log in with the following credentials: 

 

Username:  

Password:  

The WordPress dashboard displays. 

You can now create a blog post to add information to your website. 

At the top of the page, under Next Steps, choose Write your first blog post. 

If you see Welcome to the Block Editor, click Close. 

Enter a Title and write some text. Be creative! 

After you have finished, at the top-right corner of the page choose Publish... 

Choose Publish again to save your blog post. 

Task 2: Create an Amazon S3 bucket with static website hosting 

In this task, you create an Amazon S3 bucket and configure it for static website hosting. This makes the bucket accessible on the Internet via a URL. 

In a later step, you use this bucket to host a static (unchanging) version of your WordPress website. 

Return to the browser tab containing the AWS Management Console that you opened at the start of this lab. 

If you cannot find the correct browser tab, choose the Open Console button on the left of these instructions to open a new tab with the AWS Management Console. 

On the Services menu, choose S3. 

Choose Create bucket and then configure: 

 

For Bucket name, enter  

Replace NUMBER with a random number 

 

De-Select Block all public access option, and then leave all other options deselected. 

Notice all of the individual options remain deselected. When deselecting all public access, you must then select the individual options that apply to your situation and security objectives. In a production environment, it is recommended to use the least permissive settings possible. 

Select I acknowledge that ... 

Choose Create bucket 

Amazon S3 bucket names must be globally unique. If you receive an error stating The requested bucket name is not available, choose the first Edit link, change the bucket name, and try again until it works. 

Choose the link for the name of the bucket you created. The bucket overview window opens. 

Choose the Properties tab. 

Scroll down to Static website hosting. 

Select Edit 

 

Select Enable  

Select Host a static website  

For Index document, enter  

For Error document, enter  

Choose Save  

 

Scroll down to Static website hosting dialog box: 

Copy the Endpoint to your text editor for later use. 

It will look similar to: http://wordpress-jb92.s3-website-us-west-2.amazonaws.com 

Your Amazon S3 bucket is now ready to receive content from your WordPress website. 

Task 3: Generate a static version of WordPress 

In this task, you use the wp-static utility to generate a static copy of your WordPress website as HTML pages. These pages will contain a copy of the entire website and can be used without the WordPress server. 

For more information about the wp-static utility, refer to the Additional resources section at the end of this guide. 

Open the AWS Management Console. 

On the Services menu, choose EC2. 

In the left navigation pane, click Instances. 

On the Amazon EC2 dashboard, choose Connect 

In the Connect to your instance window, select EC2 Instance Connect (browser-based SSH connection). Leave the default User name as and choose Connect 

A new window opens with an SSH session to the instance. 

In the EC2 Instance Connect session, enter the following command to configure the Apache web server to allow permlinks override: 

sudo sed -i.bak -e 's/AllowOverride None/AllowOverride All/g' /etc/httpd/conf/httpd.conf; 
 

Enter the following command to restart the Apache web server to apply the permlinks override change: 

sudo service httpd restart 
 

Enter the following commands to download the wpstatic tool and generate a static HTML version of the WordPress web site: 

cd /var/www/html/wordpress; 
sudo wget https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-39/4.1.5.prod/scripts/wpstatic.sh; 
sudo /bin/sh wpstatic.sh -a; 
 
  

Many lines should be displayed in your SSH window. If this did not happen, you might need to press ENTER to run the last line of the commands. 

Enter the following command to make the Apache webserver user account the owner of the files created by the wpstatic.sh script in /var/www/html/wordpress/: 

sudo chown -R apache:apache /var/www/html/wordpress 
  

You now have a static version of your website! It can be viewed in your web browser. 

Copy the StaticURL address shown to the left of these instructions. 

Paste the URL into a new web browser tab, and then press Enter. 

Your WordPress website displays. Examine the page URLs and notice that it is now being served as static HTML pages. WordPress is not involved in serving these static pages. 

Task 4: Uploading static WordPress pages to Amazon S3 

In this task, you use the AWS Command Line Interface (AWS CLI) to copy the static pages you generated in the previous task to Amazon S3. You are then able to access the website even when the WordPress web server is turned off. 

This has several advantages: 

Amazon S3 is highly scalable and can service more users than a single web server. 

Serving content from Amazon S3 does not require any Amazon EC2 instances to be running, thereby lowering costs. 

 

Enter the following commands to retrieve the name of AWS region and the name of your Amazon S3 bucket and store the values in variables: 

# Determine Region 
AZ=`curl --silent http://169.254.169.254/latest/meta-data/placement/availability-zone/` 
REGION=${AZ::-1} 
 
# Retrieve Amazon S3 bucket name starting with wordpress-* 
BUCKET=`aws s3api list-buckets --query "Buckets[?starts_with(Name, 'wordpress-')].Name | [0]" --output text` 
  

By running these commands, you have stored the AWS Region of the EC2 instance in a variable named AZ. You have also used an AWS CLI command to retrieve any Amazon S3 bucket names in your account with names that start with wordpress- and stored the result in a variable named Bucket. 

Enter the following command to copy the WordPress static files to your Amazon S3 hosted website: 

aws s3 sync --acl public-read /var/www/html/wordpress/wordpress-static s3://$BUCKET 
  

In the previous task, you used the wp-static command to generate static HTML pages of your WordPress site. By running this command you have copied the static files from the wordpress-static directory to the Amazon S3 bucket you created. Notice the command uses the Bucket variable from the previous command. 

Your website is now available in Amazon S3! 

Retrieve the Endpoint that you previously saved into a text editor. It should look something like: http://wordpress-jb92.s3-website-us-west-2.amazonaws.com  

Copy and paste the endpoint URL into a new browser tab to view your static WordPress site hosted from Amazon S3. 

If you cannot find the endpoint, return to the web browser tab with the S3 console, choose Static website hosting and then choose the Endpoint that is displayed. 

Task 5: Using scripts to upload changes to Amazon S3 

Now that you have created a static version of your website and have uploaded changes, you may want to simplify this operation in the future. In this task, you create a script to update the static files with any changes from WordPress and upload the changes Amazon S3 to replace running each command manually. 

Enter the following commands to create a new shell script that extracts the pages from WordPress and copies them to your Amazon S3 bucket: 

echo "cd /var/www/html/wordpress; sudo rm -rf wordpress-static; sudo /bin/sh wpstatic.sh -a; aws s3 sync --acl public-read --delete /var/www/html/wordpress/wordpress-static s3://$BUCKET" > $HOME/wordpress-to-s3.sh; 
 
chmod 0755 $HOME/wordpress-to-s3.sh; 
  

Below is a breakdown of each command in the script, with comments to clarify what each command does: 

# Change to the wordpress directory 
cd /var/www/html/wordpress 
 
# Delete the wordpress-static directory 
sudo rm -rf wordpress-static 
 
# Run the wp-static utility to create static HTML pages of the current WordPress site 
sudo /bin/sh wpstatic.sh -a 
 
# Copy the WordPress static files to the S3 bucket 
aws s3 sync --acl public-read --delete /var/www/html/wordpress/wordpress-static s3://$BUCKET 
  

You can now add a new page to WordPress and then execute the script to confirm that the content is being copied to Amazon S3. 

Return to your web browser tab that is running WordPress. If you cannot find the tab, copy the WordPressURL shown to the left of these instructions and paste it into a new browser tab. 

Choose the New link at the top of the page to create a new blog post. 

Enter a title and some text, choose Publish..., and then choose Publish again. 

Return to the SSH session and enter the following command to execute the script you created previously: 

/home/ec2-user/wordpress-to-s3.sh 
 

Return to your browser tab that is connected to your Amazon S3 bucket and refresh the page. Your new blog post is displayed on the page. If it does not appear, wait one minute and refresh the browser page. 

If you cannot find the tab, retrieve the Endpoint that you earlier saved into a Text Editor. It should look something like: http://wordpress-jb92.s3-website-us-west-2.amazonaws.com 

If you cannot find the Endpoint, return to the web browser tab with the S3 Management Console, choose Static website hosting and choose the Endpoint that is displayed. 

Some things to consider 

Turning off the Amazon EC2 instance: Once your static pages have been copied to Amazon S3, you could stop your Amazon EC2 instance to save money. Simply start it again when you wish to write another blog post, then turn it off once the updates have been copied to Amazon S3. 

A friendly URL: The URL to your Amazon S3 hosted website is not easy to remember. You could create a more friendly custom domain name using Amazon Route 53. 

Adding dynamic content to pages: While WordPress is an interactive application, pages stored in Amazon S3 are static and cannot respond to user input. If you want dynamic content on pages, consider using Javascript to add logic within the web browser, or use plugin services such as Disqus to host content from a different website. 

Conclusion 

Congratulations! You have now successfully: 

Configured WordPress on Amazon EC2. 

Exported WordPress to static files. 

Copied static files to an Amazon S3 static website. 

End Lab 

Follow these steps to close the console, end your lab, and evaluate the experience. 

Return to the AWS Management Console. 

On the navigation bar, choose awsstudent@<AccountNumber>, and then choose Sign Out. 

Choose End Lab 

Choose OK 

(Optional): 

 

Select the applicable number of stars  

Type a comment 

Choose Submit 

 

1 star = Very dissatisfied 

2 stars = Dissatisfied 

3 stars = Neutral 

4 stars = Satisfied 

5 stars = Very satisfied 

sudo sed -i.bak -e 's/AllowOverride None/AllowOverride All/g' /etc/httpd/conf/httpd.conf;

sudo service httpd restart

```powershell
cd /var/www/html/wordpress;
sudo wget https://us-west-2-aws-training.s3.amazonaws.com/awsu-spl/spl-39/4.1.5.prod/scripts/wpstatic.sh;
sudo /bin/sh wpstatic.sh -a;
```

sudo chown -R apache:apache /var/www/html/wordpress
