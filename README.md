# URL Checker

### Description
Shell script to check List of URLs using CURL and test for response
Can be used to check long list of URLs continuously every few minutes and send email notification in case of any URL failures.

### Features
* Can curl the URLs and check different Reponses for different URLs
* Can test common multiple set of reponses for each URL
* Sends emails using email template using just telnet without any additional library configuration
* In case of any URL failure sends email only every 30/60 etc minutes instead of every 1 minute even if the task runs every 1 minute.
* Sends email in case of change of number of failed urls immediately.

### Usage
Set the URLs in the array in the script and save as a shell filed, set execution flag to the file and execute the shell script
Set is as a crontab to run more frequently (Recommeded)

### Source
```bash
#File to store the details
FILE="/home/maxadmin/checkurls.txt"

#SMTP Server Details for sending emails
SMTPSERVER="10.13.15.15"
SMTPSERVERPORT="25"

#Email Template file containing the above text that will be replaced
MAILTEMPLATEFILE="/home/maxadmin/mailtemplate.txt"

#Email reaplcement text to look for in the mail template file to replace for sending email
EMAILREPLACEMENTSTRING="EMAILGOESHERE"

#Response received from MAil server to establish if email was sent successfully
EMAILSUCCESSRESPONSE="Queued mail for delivery"

#Minimum time between sending email notification for same number of failures.
minTimeBetweenEmails=3600

#Minimum number of failures to take any action
minFailuresForAction=0

#Array of URLs comma seperated by the Name of the URL. 
# SERVER_NAME;URL;RESPONSE_TEST_STRING

declare -a arr=( "Main Server URL URL;http://server.name.com/makeworldabetterplace/"
                "Web Server1;http://10.11.12.13/makeworldabetterplace/" 
                "Web Server2;http://10.11.12.14/makeworldabetterplace/" 
                "Server 1 REPORTServer1-01;http://10.11.12.13:9081/makeworldabetterplace/" 
                "Server 1 APPServer01-01;http://10.11.12.13:9082/makeworldabetterplace/" 
                "Server 1 APPServer01-02;http://10.11.12.13:9083/makeworldabetterplace/" 
                "Server 1 APPServer01-03;http://10.11.12.13:9084/makeworldabetterplace/" 
                "Server 1 APPServer01-04;http://10.11.12.13:9085/makeworldabetterplace/" 
                "Server 1 APPServer01-05;http://10.11.12.13:9086/makeworldabetterplace/" 
                "Server 1 APPServer01-06;http://10.11.12.13:9087/makeworldabetterplace/" 
                "Server 1 APPServer01-07;http://10.11.12.13:9088/makeworldabetterplace/" 
                "Server 1 APPServer01-08;http://10.11.12.13:9089/makeworldabetterplace/" 
                "Server 1 APPServer01-09;http://10.11.12.13:9090/makeworldabetterplace/" 
                "Server 2 REPORTServer2-01;http://10.11.12.14:9081/makeworldabetterplace/" 
                "Server 2 APPServer02-01;http://10.11.12.14:9082/makeworldabetterplace/" 
                "Server 2 APPServer02-02;http://10.11.12.14:9083/makeworldabetterplace/" 
                "Server 2 APPServer02-03;http://10.11.12.14:9084/makeworldabetterplace/" 
                "Server 2 APPServer02-04;http://10.11.12.14:9085/makeworldabetterplace/" 
                "Server 2 APPServer02-05;http://10.11.12.14:9086/makeworldabetterplace/" 
                "Server 2 APPServer02-06;http://10.11.12.14:9087/makeworldabetterplace/" 
                "Server 2 APPServer02-07;http://10.11.12.14:9088/makeworldabetterplace/" 
                "Server 2 APPServer02-08;http://10.11.12.14:9089/makeworldabetterplace/" 
                "Server 2 MXUIServer02-09;http://10.11.12.14:9090/makeworldabetterplace/" 
                "APP Server Console;https://10.11.12.13:9043/ibm/console/;HTTP/1.1 200 OK"
                )

#Gets the value from the output file
getValue()
{
	#echo "getValue() Get Value Called with param=$1"
	if test -f "$FILE"; 
	then 
		#echo "getValue() $FILE exists"
		#echo "command = grep -o -m 1 -e '$1.*' $FILE"
		if grep -q "$1" $FILE ;
		then
			lastFailTimex=$(grep -o -m 1 -e "$1.*" $FILE)
			#echo "grep output $lastFailTime"
			lastFailTimex=$(echo $lastFailTimex | sed "s/$1//")
			#echo "sed output $lastFailTime"
			#echo "getValue() >>>>>>$lastFailTime<<<<<<<<"
			echo "$lastFailTimex"
		else
			echo ""
		fi
		
		#returnValue="$lastFailTime"
	else
		#returnValue=""
		echo ""
	fi
	#echo "getValue() exit"
}

#Send email using the mail template file.
# The mail template file contains the telnet commands to send email with a replacement text
sendEmail(){
	#echo "Param = $1"
	testReplacement="$1"
	#echo 'cat "mailtemplate.txt" | sed "s/EMAILGOESHERE/$testReplacement/"'
	emailBody=$(cat "$MAILTEMPLATEFILE" | sed "s|$EMAILREPLACEMENTSTRING|$testReplacement|")
	#echo "$emailBody"
	emailResp=$(echo "$emailBody" | nc --crlf "$SMTPSERVER" "$SMTPSERVERPORT")
	#echo "$emailResp"
	if echo "$emailResp" | grep -q "$EMAILSUCCESSRESPONSE" ;
	then
		echo "Email Sent successfully."
	else
		echo "Email Could not be sent."
	fi
}

updateAndSendEmail(){
	currenttime=$(date)
	newText="==============================\n"
	newText+="Last Failure:$currenttime\n"
	newText+="Total Failures:$1\n"
	newText+="URLs that failed Checks:\n"
	newText+="$2"
	newText+="==============================\n"
	echo -e "$newText" > "$FILE"
	echo "File Updated."
	#emailBody=$(cat "mailtemplate.txt" | sed "s/EMAILGOESHERE/$newText/")
	#emailResp=$(echo "$emailBody" | nc --crlf 10.192.97.84 25)
	sendEmail "$newText"
}

## Common Validations for all URLs if validation string is not specified as 3rd delimited string
declare -a validations=("welcome=true" "HTTP/1.1 200 OK" )
failureCount=0
## now loop through the above array
for urldetails in "${arr[@]}"
do

   #Splits the text into Names and URLs
   urlNames=$(echo "$urldetails" | cut -d ";" -f 1)
   siteUrls=$(echo "$urldetails" | cut -d ";" -f 2)
   validatestring=$(echo "$urldetails" | cut -d ";" -f 3)
	
	 #resoutput=$(curl --max-time 10 --head --silent --stderr - "$siteUrl" 2>&1)
	 resoutput=$(curl --insecure --max-time 20 --location --head "$siteUrls" 2>&1)
	
	#Check if validatestring variable is empty
	if [ -z "$validatestring" ] 
	then
		#echo "Validate string $validatestring is null"
		for validation in "${validations[@]}"
		do
			if ! echo "$resoutput" | grep -q "$validation" ;
			then
				failures+="[$urlNames] ($siteUrls)\n"
				failureCount=$(expr $failureCount + 1)
				break
			fi
		done
	else
		#echo "Validate string $validatestring is present"
		if ! echo "$resoutput" | grep -q "$validatestring" ;
			then
				failures+="Failed [$urlNames] ($siteUrls)\n"
				failureCount=$(expr $failureCount + 1)
		fi
	fi
#End of looping each URL
done

if [ "$failureCount" -gt "$minFailuresForAction" ]
then
	printf "$failures"
	
	
	if test -f "$FILE"; 
	then
		echo "$FILE exists."
		# Read File 
		# Get Last Failure Time
		prevFailureCounts=0
		newLastFileTime=$(getValue "Last Failure:")
		prevFailureCounts=$(getValue "Total Failures:")
		echo "Previous Failures=$prevFailureCounts"
		lastFailTimeEpoch=$(date --date "$newLastFileTime" +%s)
		
		currentFailTimeEpoch=$(date +%s)
		echo "Old Time=$lastFailTimeEpoch"
		echo "Now Time=$currentFailTimeEpoch"
		diffTime=$(expr $currentFailTimeEpoch - $lastFailTimeEpoch )
		echo "Diff Time = $diffTime"
		
		if [ "$diffTime" -gt "$minTimeBetweenEmails" ] ;
		then
			echo "Last email sent, over $minTimeBetweenEmails seconds, updating file and sending email again"
			updateAndSendEmail "$failureCount" "$failures"

		else
			echo "Last email sent less than $minTimeBetweenEmails seconds back"
			#Check if failures increased. if so send another email.
			failureCount=$(( $failureCount + 0 ))
			prevFailureCounts=$(( $prevFailureCounts + 0 ))
			echo " prevFailureCounts= $prevFailureCounts failureCount= $failureCount"
			if [ "$failureCount" == "$prevFailureCounts" ];
			then
				echo "Failures have not changed. Old= $prevFailureCounts New= $failureCount Therefore not sending email again."
			else
				echo "Failures have changed. Old=$prevFailureCounts New=$failureCount Sending new email."
				updateAndSendEmail "$failureCount" "$failures"
			fi
		fi
	else
		echo "$FILE doesn't exist. Will Create a new one."
		# Create new file and save the URL failed and Time
		updateResp=$(updateAndSendEmail "$failureCount" "$failures")
		
	fi

	# Update File for failure with date
	# If date in file is less than 30 mins old then don't send email
	# If no failure in file send email
	
else 
	echo "All URLs verified successfully."
	if test -f "$FILE"; 
	then
		echo -e "" > "$FILE"
		echo "$FILE exists. File Cleared"
	else
		echo "$FILE not found."
	fi
fi


```
### Pending changes
* Make email sending better instead of telnet mail template
* Handle authentication for SMTP for servers that require
* Send fancier emails
