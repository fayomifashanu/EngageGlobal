# EngageGlobal


Table of Contents

# Overview
* Production Infrastructure:\
* Continuous Integration\
* ADHOC Scripts\
* Support Procedure\
..* List of authenticated contacts:\
..* Access credentials:\
..* Detailed support procedure:\
* Known Issues\
\
\
# Overview\
AWS account number:\
Console URL:\
\
\
Environment type (production/staging/development): 

Production Infrastructure:
VPC
eng vpc-0de7b269
Subnets
eng1 subnet-5f197407
eng2 subnet-cc1434a8
eng3 subnet-1a9ac06c
Security Groups
eng-elb-sg sg-13f59775
eng-web-sg sg-c1f597a7
eng-rds-sg sg-b3f597d5
eng-geo-elb-sg sg-af7631c9
eng-geo-web-sg sg-3a75325c
pixpro-ssh-http-access sg-9ffb99f9
db-ssh-http-access sg-61fa9807
ELB
eng-elb-408209066.eu-west-1.elb.amazonaws.com
eng-geo-elb-1956647838.eu-west-1.elb.amazonaws.com

RDS eng-rds.cxseqljtf8kz.eu-west-1.rds.amazonaws.com

S3
eng-media
eng-edocman
enggeo-media
enggeo-edocman

Continuous Integration
The developers didn't want a cron job to run a svn checkout every few minutes. Instead they want to ssh on to the server and run a checkout. To allow this to be fluid, I've got two scripts running - inotify.sh to check changes to the svn folder and image.sh to create new image/launch config if there is a change. Please see Adhoc scripts.
I have advised them that using EFS would be a better option, but they do not wish to use that at this stage.

ADHOC Scripts
All scripts are located at /scripts


testip.sh
This script runs every 15 minutes on each instance. It checks that the insatnce has an Elastic IP and if not it associates one of the free ips. This is ensure when a server is repalced by autoscaling, that it has one of the static IPs. 

	#!/bin/bash
	set -x
 
	#Does instance currently have an EIP
	EC2_INSTANCE_ID=$(ec2metadata --instance-id)
	  if [[ -z `aws ec2 describe-addresses | grep ${EC2_INSTANCE_ID}` ]]; then
	    CURRENT_IP=`curl icanhazip.com`
	    FREE_IP=$(aws ec2 describe-addresses --query 'Addresses[?InstanceId==null]' | grep PublicIp | gawk '{print$2}' | sed s/\"//g | sed s/,//g)
	    ALLOC_ID=$(aws ec2 describe-addresses --output text | grep ${FREE_IP} | gawk '{print $2}')
	    aws ec2 associate-address --instance-id $EC2_INSTANCE_ID --allocation-id ${ALLOC_ID}
	  else
	    echo "ALL GOOD"
	  exit
	  fi
	exit
</code bash>
 
\\
**inotify.sh**  
This script is set to continually run after the instance starts. Inside /etc/rc.local is the following line:
/bin/sh /scripts/inotify.sh >> /scripts/inotify_output.  
\\
The script itself monitors the /var/www/html/.svn directory for any changes. If there are any changes then the /scripts/inotify_output file is appended to.  
\\
<code bash>
	#!/bin/bash
	set -x
 
	DIR=/var/www/html/.svn
	/usr/bin/inotifywait -q -m -r -e modify,attrib,close_write,move,create,delete $DIR
</code bash>
\\
\\
**image.sh**  
 
This script runs every five minutes and checks the inotify_output file to see if it has been modified within the last five minutes. If it has then this means that the /var/www/html/.svn directory has been updated - this will happen after every svn checkout. (see the /scripts/inotify.sh script). If a change to that directory has been made then this script takes a new image of the instance, creates a new Launch Configuration and updates the AutoScaling Group.  
\\
<code bash>
#!/bin/bash
	set -x
 
	# Input file
	FILE=/scripts/inotify_output
	# How many seconds before file is deemed "older"
	OLDTIME=300
	# Get current and file times
	CURTIME=$(date +%s)
	FILETIME=$(stat $FILE -c %Y)
	TIMEDIFF=$(expr $CURTIME - $FILETIME)
	DATE=$(date +"%Y%m%d_%H%M%S")
	ASG=eng-web-asg
	SGS="sg-61fa9807 sg-9ffb99f9 sg-c1f597a7"
	NAME=eng-web
	KEY=eng-ssh
	# Check if file older
	if [ $TIMEDIFF -lt $OLDTIME ]; then
	  INSTANCE=$(aws ec2 describe-instances --query 'Reservations[*].Instances[*].[State.Name, InstanceId]' --filters "Name=tag:Name,Values=$NAME" --output text | grep 'running\|pending' | gawk '{print $2}' | head -1)
	  aws ec2 create-image --instance-id $INSTANCE --name "NAME-$DATE" --description "$NAME-$DATE" --no-reboot
	  sleep 120
	  AMI=$(aws ec2 describe-images --owners self --filter 'Name=name,Values=eng-web*' | jq '.[]|max_by(.CreationDate)|.ImageId' | sed -e 's/^"//' -e 's/"$//')
	  aws autoscaling create-launch-configuration --launch-configuration-name $NAME-$DATE-LC --key-name $KEY --security-groups $SGS --image-id $AMI --instance-type t2.small
	  aws autoscaling update-auto-scaling-group --auto-scaling-group-name $ASG --launch-configuration-name $NAME-$DATE-LC
	  echo "New image created $AMI for Engage Global" | mail -s "Engage Update" paul.blake@databarracks.com < /dev/null
	fi
	exit
</code bash>
\\
\\
**cronjob to check crashed mysql tables and repair them. Runs 01.00 nightly and only runs on the instance that is first in the ASG**  
\\
 
<code bash>
#!/bin/bash
#Description : Script ensures task is ran on the first instance in the auto scaling group only
set -x
PATH=/root
INSTANCE_ID=`curl -s http://169.254.169.254/latest/meta-data/instance-id`
 
ASG=$(aws autoscaling describe-auto-scaling-instances --instance-ids $INSTANCE_ID --output text | awk '{print $2}')
 
ASG_INSTS=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-name $ASG --output text | grep "INSTANCES" | awk '{print $4}')
 
FIRST=$(aws ec2 describe-instances --instance-ids ${ASG_INSTS[@]} --query 'Reservations[*].Instances[*].[InstanceId,LaunchTime]' --output text | sort -k2 | head -n1 | cut -f1)
 
if [ "$INSTANCE_ID" == "$FIRST" ]; then
        /usr/bin/mysqlcheck --defaults-file=/root/.my.cnf --auto-repair --check --all-databases -heng-rds.cxseqljtf8kz.eu-west-1.rds.amazonaws.com > /root/mysqlcheck.txt
else
        echo 0
fi
</code bash>
 
 
 
 
#Monitoring
Logic Monitor check the following URLs:  
atcportal.net  
www.atcportal.net  
atcgeoportal.net  
www.atcgeoportal.net  
\\
CloudWatch alerts:  
eng-web-asg-high-cpu  
eng-elb-healthy-hosts  
\\
\\
To monitor memory usage and disk space I will install a perl script as per these instructions:  
\\
<code>
#Download the following packages:
 
sudo yum install perl-Switch perl-DateTime perl-Sys-Syslog perl-LWP-Protocol-https perl-Digest-SHA -y
 
sudo yum install zip unzip
 
#Download and configure the script
 
curl http://aws-cloudwatch.s3.amazonaws.com/downloads/CloudWatchMonitoringScripts-1.2.1.zip -O
unzip CloudWatchMonitoringScripts-1.2.1.zip
rm CloudWatchMonitoringScripts-1.2.1.zip
cd aws-scripts-mon
 
#Create an IAM role with following permissions:
 
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:PutMetricData",
        "ec2:DescribeTags"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
 
#Test 
 
./mon-put-instance-data.pl --mem-util --verify --verbose --aws-access-key-id xxxxxxxxxx --aws-secret-key xxxxxxxx
 
#Create cron job to run every five minutes
 
*/5 * * * * ~/aws-scripts-mon/mon-put-instance-data.pl --aws-access-key-id xxxxxxxxxxxxxxxxxxx --aws-secret-key xxxxxxxxxxxxxxxxxxxx --mem-util --disk-space-util --disk-path=/ --from-cron
