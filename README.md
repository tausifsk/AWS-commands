Find out S3 Lifecyle details :
aws s3api list-buckets --query "Buckets[].Name" --output text | tr '\t' '\n' | while read line ; do lf=`aws s3api get-bucket-lifecycle-configuration --bucket "${line}" --query "Rules[*].Status" --output text 2>&1` ; echo "$line --- $lf"; done

Getting S3 zise
aws s3api list-buckets --query "Buckets[].Name" --output text | tr '\t' '\n'  | head -10 | while read line ; do sz=`aws s3 ls s3://"${line}" --recursive --human-readable --summarize | awk END'{print}'` ; pr=`aws s3api get-bucket-tagging --bucket ${line}" --query "TagSet[?Key=='Project'].Value | [0] --output text"` ; echo "$line # $sz # pr"; done

Find All OLD AMI –

List All Ami 30 days older---
aws ec2 describe-images --owners self --query "Images[?(CreationDate<'`date +%F -d '30 day ago'`')][CreationDate,ImageId,ImageType,PlatformDetails,sum(BlockDeviceMappings[*].Ebs.VolumeSize),Description,Name,Tags[?Key=='Name'].Value | [0]]" --output table | tr -d " "  > ami.txt

List all instance-----------

aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,ImageId]" --output table | tr -d " " > ec2.txt

List all template ---------
    aws ec2 describe-launch-templates --query "LaunchTemplates[*].LaunchTemplateId" --output text | tr '\t' '\n' | head | while read line ; do aws ec2 describe-launch-template-versions --launch-template-id $line --query "LaunchTemplateVersions[*].[LaunchTemplateId,LaunchTemplateData.ImageId,VersionNumber,DefaultVersion]" --output table | tr -d " " >> template.txt ; done
    touch template.txt && cat template.txt >> ec2.txt

Find all Ami that’s associated/Not associated with Any Ec2 , template 
    awk -v acc=$account_id -v acc_name=$account_name -F"|" 'NR==FNR { gsub($3," ","") ; if ( $3 in out ) {out[$3]=out[$3] "#" $2 "-" $4 "-" $5} else {out[$3]=$2 "-" $4 "-" $5} ; next ;} { if ($3 in out) {print acc "|" acc_name "|" $0 "|" out[$3] "|Inuse"} else { print acc "|" acc_name "|" $0 "|NONE|" "Not in use"} }' ec2.txt ami.txt > out.csv

Cloudwatch Logs Incoming Byte and Number of calls ( need to consider IncomingLogEvents & IncomingBytes change period , start , stop date )
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
aws cloudwatch list-metrics  --namespace "AWS/Logs" --query "Metrics[*].[MetricName , Dimensions[?Name=='LogGroupName'].Value | [0]]" --output text | while read line ; do met_name=`echo $line | awk '{print $1}'`; val=`echo $line | awk '{print $2}'` ; aws cloudwatch get-metric-statistics --namespace AWS/Logs --metric-name $met_name --start-time 2023-08-24T00:00:00Z  --end-time 2023-09-07T00:00:00Z --period 86400 --statistics Sum --dimensions Name=LogGroupName,Value=$val --query "Datapoints[*].[ Timestamp , Sum ]" --output text | awk -v met_name=$met_name -v val=$val '{print met_name "," val "," substr($1,1,10) "," $2}'; done


Same type of command can be used for CPU utilization etc 
-----------------------------------------------------------
aws rds describe-db-instances --query "DBInstances[*].DBInstanceIdentifier" --output text | tr '\t' '\n' | xargs -I {} aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization --start-time 2023-08-22T00:00:00Z  --end-time 2023-09-22T00:00:00Z --period 36000 --statistics Maximum --dimensions Name=DBInstanceIdentifier,Value={} --output table


OLD DB snaps 
aws rds describe-db-snapshots --query "DBSnapshots[?(SnapshotCreateTime<'`date +%F -d '30 day ago'`')].[DBSnapshotIdentifier,DBInstanceIdentifier,SnapshotCreateTime,InstanceCreateTime,AllocatedStorage,Tags[?Key=='Project'].Value [0],SnapshotType,Status]" --output table


Vpc flow log 
--------------------------

aws ec2 describe-flow-logs --query "FlowLogs[*].[FlowLogId,LogGroupName,ResourceId,LogDestinationType]" --output table



Load balancer and Target group needs Vlookup in xls 
--------------------------------------------

aws elbv2 describe-load-balancers --query "LoadBalancers[*].[LoadBalancerName,LoadBalancerArn,Scheme,VpcId]" --output table
aws elbv2 describe-target-groups --query 'TargetGroups[*].[TargetGroupArn,LoadBalancerArns[0]]' --profile voe --output table

need to loop through the above output to get healthy/unhealthy targets
------------------------------------------------------------------------------------------------------
aws elbv2 describe-target-health --target-group-arn <LOOPED VALUE> --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' --output text
