# git-to-AWS-sync
Automatically Deploy from GitHub Using AWS CodeDeploy
AWS CodeDeploy is a new service that makes it easy to deploy application updates to Amazon EC2 instances. CodeDeploy is targeted at customers who manage their EC2 instances directly, instead of those who use an application management service like AWS Elastic Beanstalk or AWS OpsWorks that have their own built-in deployment features. CodeDeploy allows developers and administrators to centrally control and track their application deployments across their different development, testing, and production environments.

For a quick overview of what CodeDeploy can do, watch this introductory video. In this video, we demonstrate automatically triggering a deployment from a source code change in a GitHub repository. GitHub is a popular code management and developer collaboration tool. By connecting GitHub to CodeDeploy, you can set up an end-to-end pipeline to move your code changes from source control to your testing or production environments. The remainder of this post walks through the steps required to set up automatic deployments from GitHub to CodeDeploy.

Setting Up the Prerequisites
To start with, we’ll assume that you already have an application set up in CodeDeploy that’s successfully deploying to a set of EC2 instances. You can learn about the steps required to do this in our User Guide. To get started quickly, you can also create a sample deployment to a set of test instances through our Getting Started Wizard in the console. The steps below will use this getting started sample application, but you can also translate these actions to your own application.

Moving Your Application Into GitHub
If the application files that you want to deploy are not already in a GitHub repository, you’ll need to set that up. Here’s how you can do it with the getting started sample application. First, download the application files. These examples use Linux / Unix commands.

mkdir codedeploy-sample
cd codedeploy-sample
curl -O http://s3.amazonaws.com/aws-codedeploy-us-east-1/samples/latest/SampleApp_Linux.zip
unzip SampleApp_Linux.zip
rm SampleApp_Linux.zip
Next, you need to create a repository on GitHub to store these application files. If you need help, you can read the GitHub documentation. Your repository can be public or private. After the GitHub repository is created, you’ll push your local application files to it.

git init
git add .
git commit -m "first commit"
git remote add origin git@github.com:YOUR_USERNAME/YOUR_REPOSITORY.git
git push -u origin master
Deploying Application Files from GitHub
Once your application files are in GitHub, you can configure CodeDeploy to pull the application bundle directly from the GitHub repository, rather than from Amazon S3. Let’s trigger a deployment from your GitHub repository using the AWS Management Console. From the Deployments page, click Create New Deployment. Select the name of your application, the target deployment group, and GitHub for the revision type. You should then see a Connect to GitHub section.



Click Connect With GitHub, and then step through the OAuth process. A few different things might happen next. First, if you are not logged into GitHub in your browser, you will be asked to log in. Next, if you haven’t already granted AWS CodeDeploy access to your GitHub repositories, you will be asked to authorize that now. Once this is done, you’ll return to the AWS Management Console and CodeDeploy will have the permissions required to access your repository. All that’s left is to fill in the Repository Name and Commit ID. The repository name will be in the format “GITHUB_USERNAME/REPOSITORY_NAME”. The commit ID will be the full SHA (a 40-digit hex number) that can be copied through the GitHub UI. You can find this information from the commit history page of your repository.



Click Deploy Now, and then monitor the deployment in the console to ensure that it succeeds. After you’re sure this is set up correctly, you can proceed with configuring the automatic deployment from GitHub.

Calling AWS CodeDeploy from GitHub
There are two service hooks that you need to configure in GitHub to set up automatic deployments. The first is the AWS CodeDeploy service hook that enables GitHub to call the CodeDeploy API. When a third party requires access to your organization’s AWS resources, the recommended best practice is to use an IAM role to delegate API access to them. By allowing a partner’s AWS account to assume a role in your account, you avoid sharing long-term AWS credentials with the partner. But if the partner you want to integrate with does not yet support roles, you should create an IAM user for your application with limited permissions. We will take that approach here and use the access keys for this user when making the AWS calls from GitHub. Go to the IAM Users page in the AWS Management Console. Click Create New Users. Enter “GitHub” for the user name in the first row.



Make sure that the option to generate an access key is checked, and click Create.



On the next page, click Show User Security Credentials to show the Access Key ID and Secret Access Key for the new user. Copy these down and store them in a safe and secure location, because this screen will be your last opportunity to download the secret key.

After you have the credentials, you can close out of the wizard. Next, you need to attach a policy to the new user to give them access permissions. Click the GitHub user in the IAM Users list. On the user page, scroll down to the Permissions section, and click Attach User Policy. Select the Custom Policy option and click Select. Enter a Policy Name like “CodeDeploy-Access”, and enter the following JSON into the Policy Document. You will need to replace “us-east-1” if you are using a different region, and replace “123ACCOUNTID” with your AWS account ID that is found on your Account Settings page. This policy is crafted to give the GitHub user only the minimum permission to call the CodeDeploy service APIs required for deployment.

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "codedeploy:GetDeploymentConfig",
      "Resource": "arn:aws:codedeploy:us-east-1:123ACCOUNTID:deploymentconfig:*"
    },
    {
      "Effect": "Allow",
      "Action": "codedeploy:RegisterApplicationRevision",
      "Resource": "arn:aws:codedeploy:us-east-1:123ACCOUNTID:application:DemoApplication"
    },
    {
      "Effect": "Allow",
      "Action": "codedeploy:GetApplicationRevision",
      "Resource": "arn:aws:codedeploy:us-east-1:123ACCOUNTID:application:DemoApplication"
    },
    {
      "Effect": "Allow",
      "Action": "codedeploy:CreateDeployment",
      "Resource": "arn:aws:codedeploy:us-east-1:123ACCOUNTID:deploymentgroup:DemoApplication/DemoFleet"
    }
  ]
}
Click Apply Policy. Now you’re ready to configure the AWS CodeDeploy service hook on GitHub. From the home page for your GitHub repository, click on the Settings tab.



On the Settings page, click the Webhooks & Services tab. Then in the Services section, click the Add Service drop-down, and select AWS CodeDeploy. On the service hook page, enter the information needed to call CodeDeploy, including the target AWS region, application name, target deployment group, and the access key ID and secret access key from the IAM user created earlier.



After entering this information, click Add Service.

Automatically Starting Deployments from GitHub
Now, you’ll add the second GitHub service hook to enable automatic deployments. The GitHub Auto Deployment service is used to control when deployments will be initiated on repository events. Deployments can be triggered when the default branch is pushed to, or if you’re using a continuous integration service, only when test suites successfully pass.

You first need to create a GitHub personal access token for the Auto-Deployment service to trigger a repository deployment. Go to the Applications tab in the Personal Settings page for your GitHub account. In the Personal Access Tokens section, click Generate New Token. Enter “AutoDeploy” for the Token Description, uncheck all of the scope boxes, and check only the repo_deployment scope.



Click Generate token. On the next page, copy the newly generated personal access token from the list, and store it in a safe place with the AWS access keys from before. You won’t be able to access this token again.

Now you need to configure the GitHub Auto-Deployment service hook on GitHub. From the home page for your GitHub repository, click on the Settings tab. On the Settings page, click the Webhooks & Services tab. Then in the Services section, click the Add Service drop-down, and select GitHub Auto-Deployment. On the service hook page, enter the information needed to call GitHub, including the personal access token and target deployment group for CodeDeploy.



After entering this information, click Add Service.

Now you’ll want to test everything working together. From the home page of your GitHub repository, click the index.html in the file list. On the file view page, click the pencil button on the toolbar above the file content to switch into edit mode.



You can change the web page content any way you like, such as by adding new text.



When you’re done, click Commit changes. If your prior configuration is set up correctly, a new deployment should be started immediately. Switch to the Deployments page in the AWS Management Console. You should see a new deployment at the top of the list that’s in progress.

You can browse to one of the instances in the deployment group to see when it receives the new web page. To get the public address of an instance, click on the Deployment ID in the list deployments list, and then click an Instance ID in the instances list to open the EC2 console. In the properties pane of the console, you can find the Public DNS for the instance. Copy and paste that value into a web browser address bar, and you can view the home page.

Going Further
If you’d like to connect your repository to a continuous integration service for unit testing, and only want to auto-deploy when those tests pass, you can read more about deploying on a commit status from this GitHub Auto-Deployment blog post.

While this setup works great if you’re deploying a static website or a dynamic language web application, you’ll need a build step in between GitHub and CodeDeploy if your application uses a compiled language. For this, you can choose from a wide selection of partners who have integrated their continuous integration services with CodeDeploy.

To dive deeper into the AWS CodeDeploy service, you can find links to documentation, tutorials, and samples on our Developer Resources page. We’d love to hear your feedback and suggestions for the service, so please reach out to the product team through our forum.

