#!/usr/bin/python
import cgi, os
import cgitb; cgitb.enable()
import time
import subprocess

#handle html file uploads with python with some added features
#make sure that the MISC folder has 777 permissions

#signal to bash helper script that it is time to start tracking upload status
#f = open("/tmp/UPLOAD_STATUS", "w")
f = open("/var/www/html/MISC/UPLOAD_STATUS", "w")
f.write("1")
f.close()

#start bash helper script
#this way stops the python script from continuing until the bash script finishes
#subprocess.call(['sh', './HELPER_UPLOADER'])
#this way runs the 2 scripts in parallel
processes = []
processes.append(subprocess.Popen(['./HELPER_UPLOADER']))

form = cgi.FieldStorage()
# Get filename here.
fileitem = form['filename']

#check the first four letters of the uploaded file
#this is a basic security measure
#the messaging in this section does not work
PREFIX = fileitem.filename[0:4]
if PREFIX == "abc_":
	message = 'GO'
else:
	#this causes apache to print an internal server error but works properly
   	message = 'NO GO'
   	time.sleep(10)
	sys.exit("Peace")

# Test if the file was uploaded
if fileitem.filename:
   #strip leading path from file name to avoid
   #directory traversal attacks
   fn = os.path.basename(fileitem.filename.replace("\\", "/" ))
   #strip first four letters so as not to reveal the security scheme
   fn = os.path.basename(fileitem.filename.replace("abc_", "" ))
   #it was important that this script specified the path to avoid permissions errors
   #permissions errors seem to have hung the old script up and did not print messages
   #open('/tmp/' + fn, 'wb').write(fileitem.file.read())
   open('/var/www/html/MISC/' + fn, 'wb').write(fileitem.file.read())
   message = 'The file "' + fn + '" was uploaded successfully'
#this script also has some nice error checking here
else:
   message = 'No file was uploaded'
#signal to bash helper script that it is time to stop tracking upload status
#f = open("/tmp/UPLOAD_STATUS", "w")
f = open("/var/www/html/MISC/UPLOAD_STATUS", "w")
f.write("0")
f.close()
print """\
Content-Type: text/html\n
<html>
<body>
   <p>%s</p>
</body>
</html>
""" % (message,)

