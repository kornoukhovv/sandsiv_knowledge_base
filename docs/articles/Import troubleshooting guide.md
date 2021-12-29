#Import troubleshooting guide#

1.       The issue with import can appear in the #prodchecker channel or the client asks for assistance to check the status of the import since it’s been “in progress” for a while.

Usually ETA cannot be 15 hours like in the example above, however there are cases when the client imports more than a million records and it takes time to complete the process. In any case we need to check the status of import.
First of all check the import status in VoC Hub – Feedback – Imports to make sure the import indeed is “in progress” then check if the imported survey/s is not in Paused or Stopped state in Feedback - Surveys.
2.       To check the progress of import we need to go to Kubernetes by opening console and typing in “k9s”.

3.       Choose the needed cluster or if you automatically entered one of the clusters but you need to switch to another one press :ctx
GCP - gke_vochub-deployment_europe-west3-c_vcohub005
BM5 - kubernetes-admin@cluster.local

4.       Find the corresponding fb-tasts pod that you need. If I search for A1 Austria I enter the one on the screenshot.

5.       Select any container and press “S” to open shell.

If you want to exit from container shell you need to press “Ctrl+D” otherwise you can press “Esc” to go back to the list of pods.

6.       Type in grep "23705" /var/log/feedback/api/questionnaire_processor1.log to see the last created questionnaires from the import with ID 23705 (in this example).

You can try this command again to check if new questionnaires are created and the import process is indeed in progress.
Important note:
Errors for import process can be found in feedback.api.log or questionnaire_processor1.log files.
Keywords for search can be “ERROR”, “traceback”, “unexpected”.
I.e. grep "unexpected" /var/log/feedback/api/questionnaire_processor1.log
Example of an error in Import process:

“grep” command searches for lines not the whole log like above therefore to see the error completely you need to search through log with:
less /var/log/feedback/api/questionnaire_processor1.log
Use arrow down or PageDown buttons to scroll through the file.
7.       Inform about the found error in #feedback channel with log extraction like on the screenshot above but in text format for the team to investigate the issue that caused the error.
8.       Reprocessing of Import.
*WARNING: before reprocessing make sure the surveys to import have quarantines enabled to avoid duplicated invitations because import file will be reprocessed from scratch.*
To reprocess:
a)      Make sure you are in the needed fb-task pod, for instance fb-tasks-<enterprise>.
b)      You entered any container inside fb-tasks-<enterprise> and you are currently in root@fb-tasks-<enterprise>-bd95d4d76-hlb6n:/usr/feedback#
c)       Type in source venv/bin/activate
d)      Type in ./fbapi-mgm.py shell
e)      Type in:
enterprise = '<enterprise>' – where <enterprise> your enterprise, in our case it’s “a1austria” since we are in fb-tasks-a1austria.
f)   Type in:
import_id = <import id> – where <import id> your import id.
g)       Type in the following but before doing that make sure “kwargs” are filled correctly:
from feedback_api.imports.tasks.validation import import_process
from django.conf import settings
from feedback_api.imports.models import ImportInfo
etp_queue = '_'.join((enterprise, settings.QUEUE_IMPORT_VALIDATOR))
import_info = ImportInfo.objects.using(enterprise).get(id=import_id)
import_info.processed_row_count = 0
import_info.failed_row_count = 0
import_info.result_file = None
import_info.error_file = None
import_info.end_time = None
import_info.save()
import_process.apply_async(
   retry=True,
   queue=etp_queue,
   routing_key=etp_queue,
   retry_policy=dict(
       max_retries=3,
       interval_start=3,
       interval_step=1,
       interval_max=6
   ),
   kwargs={
       'enterprise': 'a1austria’',
       'import_info_id': 25122,
       'jwt': '<jwt token>'
       }
)
Where <jwt token> is your jwt token can be found at https://<vochub>/about
Here’s the example of the result done on CustomDemo hub for Import ID 1385:

<AsyncResult: 4da3bdcd-5ef3-4880-8f14-cacb0ed3bf2d> means the command was successfully executed.
 

