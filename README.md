Doing Exam Timetable for 350,000 students, 1000 modules, to schedule in 6-8 weeks...
The program run in minutes.
It returns a well optimised TT.

The problem

The principle used is to order modules based on the difficulty to place them, and then to fit them
into the first space it fits, given load management. This is for online exams, where no venue contraints
are applicable, but network and application load is considered.
Constraints:
No clashes are allowed,
Students are not allowed more than one paper per day,
Depending on the NQF-level, rest days are scheduled before a next paper,
load limits: modules per day, concurrent students, concurrent modules.

Some of our modules have more than 20,000 students, doing courses online. We limit the TT to not place modules larger
than a certain max, and place these manually.

The TT takes 2 types of assessments into account - essay type, and only multi-choice (MCQs).
MCQs are placed towards the end of the TT, and essays are given priority for being placed first.
This division, the dificulty to place a module, and load limits, are used to select(!) a module for placement, where it is then
placed on the first open slot.

The whole TT is scheduled over a number of days, where public holidays can be excluded, and days of the week be selected for inclusion. We generally run the TT for weekdays only, and ususally it does not fit. Then we include Saturdays...

Each day has a starting time, usually 0800, and ends usually at 20h00. The day is divided into 15min sessions. Each paper has a duration, between 1 and 4 hours. A paper of 2 hours will then occupy 8 sessions. We allow "login" and "upload" sessions for each paper - usually 1 session for login and download of the questionnaire ( for MCQs and essays ) and 2 sessions for uploading answer sheets. These answer sheets are "supposed" to be PDF's, which will be processed after submission for similarity checks and marking.

MCQs concurrent load is assumed to be all students registered, thoughout the whole duration of the paper. As the servers currently only handle 7500 concurrent students with ease, larger classes are broken into groups randomly, and allocated the next open slots, on the same day.
Our colleges are divided into 5 groups, based on number of students, and each college group is hosted on another server. 
(Currently on AWS cloud.) The TT takes this into account when evaluating load.
For essays we assume a full load during login sessions, upload sessions and the first and last session. During the progress of the paper we assume a load decrease to 25% of the student count in the middle of the paper's duration.

Papers are then scheduled, to interleave, within the load limits.
The TT is initially ran with no module limit per day, as the number of modules per day only presents support and administration issues. Once the TT fits within an acceptable time span, the number of modules per day is decreased, until a good spread of modules is observed, and it still is within the acceptable time period.

In our case, load limits are not the "bottle neck", the single factor influencing the total time span of the TT. The biggest factor is clashes. Our courses are very fluid ito curricula, and as such students can take between 1 and 25 modules. We have three exam periods per year, so usually not all papers have exams in the same period.
Students however do get 2nd oppertunities based on bussines rules, and that can inflate numbers.

Papers can be set manually before the TT is run. It can also be run for subsets of modules, and then repeated for modules not yet set. Or it can be asked to reschedule selected modules.

The interface

We also assume and interface where the admin officer can define:
- the time spans ito starting dates and ending dates, per assessment type.
- the days of the weeks to use and any public holidays to exclude.
- login and upload slots
- load limits per assessment type, per college group:
  - modules per day, students per slot, students per day (used load-testing to get the limits here).
- total load limits per day
- default duration (1-4 hours, usually 4 hours)
- Rest periods per NQF level
and some other selections like post/under grad, all students, students with admission, etc.


The data layer

The first data extraction is a list of all modules being assessed in this exam, the type of assessment, the duration, the college, and the number of students involved - the module list.

The second extraction is pairs of modules, with how many students are taking both modules - the clash matrix.

The third extraction is of module code that are equivilants, i.e. must be written together - equivalents.

Updating the TT merely sets the date and time for when the paper is written, without the login and upload sessions.

The algorithm

The process to do this is actually not very complicated - and is basically a knap-sack algorithm.
The most important input is however the weight placed on each module, defining the order of selection for scheduling.
Using multiple criteria, and weights per criteria, we have found that the single important factor is not the number of students, and not the number of clashing students, but rather the number of modules with which a module clashes.
Using only this one criteria for selection to place, provides a very well optimal TT.
We tested a limited exam offering, up to about 50 modules, against a brute force algorithm, that does calculate the most optimal solution. The knap-sack placement on clashing modules in all cases placed 50 modules within 1 day of the brute force algorithm, where typical exam time spans ran for 3 to 4 weeks. (Again, our clashes is the major headache.) Expanding the test to more modules became impractical.

order modules on number of modules it clash with / mcq_factor, descending
(The bigger the mcq factor, the later the module will be selected for placement. For Essays mcq factor=1)

For each module (ordered on weight desc) {
    If not yet placed 
        For each day available to it (based on paper type) 
            If daily load limits are exceeded goto next day (see module list for number of students)
            For each module already placed on that day 
                If clash with placed module - goto next day (see clash matrix )
            For each slot as start slot 
                For each slot the paper would use
                    If slot load exceeded, goto next start slot (per server, per paper type)
                    Total the loads, if totals exceeded, goto next start start slot
                Keep the start slot with the least overall load            
            If slot found, Place it, goto next module                   
        Report failure on module

When traversing the slots, the inner most loops, some optimization is possible.
For example, we only test a start slot if there are still enough slots after it for the whole paper for the day.
We also traverse the slots by doing all for the paper from slot 1, and then to test slot 2, we drop slot 1 and add the 
next end slot.
Also if any slot exceeds the limits, we move the next starts slot to after this one.
