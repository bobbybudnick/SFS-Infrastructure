# SFS-UPLOADER
Basic server side scripts that handle HTML uploads

HTML component
-----
form has action section which requires a script argument that does the work on server  
this script could be Javascript or PHP or in this case Python

Python component
-----
uploader checks for certain key file names for security  
uploader puts files in misc folder

Bash component
-----
start script near beginning of Python script  
use lsof or something else to check how much has been transfered  
update filename of special status file in misc to show how many megabytes uploaded  
quit Bash script while loop after timeout and write timeout to status  
quit Bash script while loop when Python script signal ends and write complete to status

Old idea for Python progress bar
-----
the fileitem variable for the form could be exported as a global variable  
this can not work because when the script switches to html there is no way back  
html is the only way to message the browser  
thus only one message can be displayed per upload action  
python requests library would bloat the server up but it could handle progress possibly  
requests library would still not allow dynamic updating through html to the uploader
     
Determining file size
-----
it may or may not be available to the server but the amount transferred always will  
so the amount transferred can be shown to the uploader somehow  
presumably the client knows the file size and can do the subtraction themselves

