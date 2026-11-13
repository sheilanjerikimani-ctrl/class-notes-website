# class-notes-website
Class Notes Hub

A simple website where students can find and download lecture notes by semester, hosted on AWS using EC2, S3 and my own VPC.

Author: Sheila Kimani Region: Europe (Stockholm) eu-north-1 Live demo: http://16.192.158.87 (the server is deleted after marking, so this link stops working)

1. The problem

As a class representative, I receive notes, assignments and past papers from lecturers as PDF and Word files. They arrive in different chats and emails, and they get buried. Students who miss a lesson, or who need to revise a unit they failed, struggle to find the right file. Everyone ends up asking the class rep again.

How this website helps: all the files are in one place. A student opens the site, picks a semester, and downloads the unit's notes in one tap, from a phone or a laptop, at any time.

2. What I built
A web page that lists units as cards, with a filter for Semester 1 and Semester 2.
Each card has download links. The files come from an S3 bucket.
The web page runs on an EC2 server inside my own network (VPC).
The server sets itself up automatically on first start using a user data script.
3. Architecture
   Student's browser
          |
          v
   Internet gateway
          |
 +--------+-----------------------------------------+
 |  My VPC  10.0.0.0/16                              |
 |                                                   |
 |  +---------------------------------------------+ |
 |  | Public subnet  10.0.1.0/24                   | |
 |  |   EC2 instance (Amazon Linux + Apache)       | |
 |  |   Security group: HTTP 80, SSH 22 (my IP)    | |
 |  +---------------------------------------------+ |
 |                                                   |
 |  +---------------------------------------------+ |
 |  | Private subnet  10.0.2.0/24 (empty, no route | |
 |  | to the internet)                             | |
 |  +---------------------------------------------+ |
 +---------------------------------------------------+

   S3 bucket: class-notes-hub-software-engineering
   Folder: public-notes/   (the PDFs and Word files)
   Student clicks Download  -->  file comes straight from S3
Part	What it is	Why it is there
VPC 10.0.0.0/16	My own private network in AWS	Keeps my project separate and under my control
Public subnet 10.0.1.0/24	Part of the VPC with a route to the internet	The web server must be reachable by students
Private subnet 10.0.2.0/24	Part of the VPC with no route to the internet	Shows how a protected area is kept (for example for a future database)
Internet gateway + route table	The door from the VPC to the internet	Makes the public subnet really public
Security group	Firewall for the instance	Opens only the ports I need
EC2 instance	The web server (Amazon Linux, free-tier type)	Serves the web page
User data script	A script that runs on first boot	Installs Apache and creates the page automatically
S3 bucket + bucket policy	File storage with a rule about who can read	Stores the notes safely and cheaply
4. How it works
A student opens http://16.192.158.87 in a browser.
The request goes through the internet gateway to my public subnet. The security group allows it on port 80.
Apache on the EC2 instance sends back the index.html page.
The student clicks Download. The browser requests the file directly from the S3 bucket, and the bucket policy allows it because the file is in public-notes/.
5. Steps I followed

Replace each screenshot below with your own. Keep the images in docs/screenshots/.

Step 1: Created the web page. I asked an AI assistant to write one index.html file (HTML, CSS and JavaScript together, no external libraries), then checked it in my browser.

Step 2: Created the S3 bucket. Bucket class-notes-hub-software-engineering, uploaded the notes into a folder called public-notes, and added a bucket policy that allows public reading of that folder only. S3 bucket Bucket policy

Step 3: Built the VPC. Created a VPC (10.0.0.0/16), a public subnet (10.0.1.0/24), a private subnet (10.0.2.0/24), an internet gateway, and a public route table with 0.0.0.0/0 pointing to the internet gateway. VPC and subnets Public route table

Step 4: Created the security group. HTTP (80) from anywhere, SSH (22) from my IP only. Security group rules

Step 5: Wrote the user data script. It installs Apache, starts it, and writes my index.html to the web folder. The full script is in user-data.sh.

Step 6: Launched the EC2 instance. Amazon Linux, free-tier eligible type, my VPC, the public subnet, auto-assign public IP enabled, my security group, and the user data pasted in (with the base64 checkbox left unticked). Running instance Instance network details

Step 7: Tested the site. Opened http://16.192.158.87and downloaded a file from S3. Live website A downloaded file

6. Why we combined the services

1. Why did we put the PDFs in S3 and not on the EC2 server? EC2 is for running the web server, and S3 is built for storing files. If the notes were on the server's disk, they would be lost when the instance is terminated, and every new server would need the files copied over again. In S3 the files are stored safely and cheaply, and they stay available even if the server is replaced or deleted.

2. What makes a subnet public, and what is the private subnet for? A subnet is public when its route table has a route to an internet gateway (0.0.0.0/0 to the gateway). My web server sits there so students can reach it. The private subnet has no route to the internet, so nothing from outside can reach it directly. It is where protected things, like a database, would go in a bigger project.

3. Why is SSH allowed only from My IP, but HTTP from anywhere? HTTP (port 80) is how students view the website, so it must be open to everyone. SSH (port 22) is how an administrator logs in to control the server. If it were open to the whole internet, bots would try to break in. Allowing only my IP means only I can try to log in.

4. Why does the bucket policy use public-notes/* and not the whole bucket? The /* means "every object under this path". Using public-notes/* makes only that folder readable by the public. Anything else I store in the bucket stays private. This is called least privilege: give access only to what is needed.

5. Why is user data better than typing the setup commands by hand? User data runs automatically the first time the server starts, so the server sets itself up with no one logging in. It gives the same result every time, it is faster, it avoids typing mistakes, and it is saved as a file in GitHub so anyone can repeat my setup.

7. Problems I hit and how I fixed them

Edit this table so it lists only what really happened to you. Add anything new.

Problem	Cause	Fix
Subnet creation failed: "CIDR conflicts with another subnet"	I had used the VPC wizard, which created large /20 subnets that overlapped my /24 ones	Removed that VPC and created a new one using VPC only, then made my own subnets
Download buttons did not work	The link had s3.eu-north.amazonaws.com but the Region is eu-north-1	Corrected the address in the code
Some files returned "NoSuchKey" or Access Denied	A few file names had a trailing space	Removed the spaces in the names

Unsure about the "already base64 encoded" box	It is for text you encoded yourself	Left it unticked and pasted the script as plain text
8. Clean-up (so I stay in the free tier)
 Terminated the EC2 instance
 Deleted the security group
 Deleted the VPC (subnets, route table and internet gateway went with it)
 Emptied and deleted the S3 bucket
 Deleted the key pair and my .pem file
 Checked the Billing page for running resources

Terminated instance

9. What I learned
How a VPC, subnets, an internet gateway and a route table work together to make a server reachable.
Why storage (S3) and compute (EC2) are kept separate.
How a bucket policy limits access to a single folder.
How user data automates a server setup.
That small details, such as a wrong Region name or a trailing space in a file name, can break a whole project, and how to find them.

Next steps: keep the list of notes in a separate data file so I do not edit the page for every upload, add HTTPS, and limit access to my classmates only.

Repository contents
File	Purpose
index.html	The web page
user-data.sh	The script that sets up the server
docs/screenshots/	Screenshots of each step
.gitignore	Stops key files (*.pem) from being uploaded

Security note: no keys, passwords or .pem files are stored in this repository.
