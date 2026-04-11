Secure Real-Time Voting System
QNX Neutrino RTOS
Team Genz Coders

A secure, real-time electronic voting booth built on QNX Neutrino RTOS using C. Designed for embedded systems where timing, integrity, and concurrency are critical. Everything runs on a single QNX console with no external terminals or pipes required.


ABOUT THE PROJECT

This is a real-time voting system that allows voters to cast votes securely through a console interface. The system uses QNX Neutrino RTOS features like native IPC message passing, real-time thread scheduling, and priority-inherit mutexes to guarantee correctness even under concurrent access. It was built as a hackathon project to demonstrate real-time operating system concepts in a practical, security-focused application.


HOW TO RUN

1. Open the QNX Momentics IDE
2. Import the project or copy voting.c into your workspace
3. Build the project using the QNX toolchain with the pthread library linked
4. Deploy and run the binary directly on the QNX Neutrino target console
5. The program starts automatically on the console 


HOW THE SYSTEM WORKS

Step 1 - Admin Login
   The system starts with an admin login screen.
   The admin enters the password to access the system.
   If the wrong password is entered 3 times, the system locks and displays a 10-second countdown before allowing another attempt.

Step 2 - Admin Menu
   After login the admin can add candidates, remove candidates, view current results, check live vote count, and start the voting session.
   Voting cannot begin until at least one candidate is added.

Step 3 - Voter Identification
   Each voter enters their unique Voter ID (101 to 117).
   The system checks if the ID exists in the database.
   If the voter has already voted, a fraud lockdown is triggered and the admin must enter the password to unlock the system.

Step 4 - Face Verification
   The voter looks at the camera.
   The system performs a biometric face scan (simulated via console input in this version).
   If face verification fails, access is denied and the voter cannot proceed.

Step 5 - Door Closes
   Once identity is confirmed, the door closes and the voter is inside the booth.

Step 6 - Candidate Selection
   The voter sees the list of candidates and selects one by entering the number.
   The vote is sent to the counter thread via QNX IPC message passing.
   The counter thread validates and records the vote atomically.

Step 7 - Vote Confirmation
   The console displays a success message with the updated total vote count.
   If something is wrong the vote is blocked and a reason is shown.

Step 8 - Rate Limit Check
   After a successful vote the system checks the rate limit.
   If more than 3 votes are cast within 60 seconds, the booth locks for 10 seconds with a live countdown displayed on the console.

Step 9 - Exit Verification
   The voter looks at the camera again for exit face verification.
   If it passes, the door opens and the voter exits.
   If it fails, the door stays locked and the admin is alerted.

Step 10 - Next Voter or End
   The admin can type "end" followed by the admin password to stop voting early.
   Otherwise the system loops back to the voter ID screen for the next voter.
   Voting automatically ends when the 20-minute timer runs out.

Step 11 - Final Results
   Once voting closes, the final results are displayed on the console showing each candidate's votes, percentage, and the winner.
   All events are saved to the audit log at /tmp/vote_audit.log


SECURITY FEATURES

Duplicate vote prevention
   Every voter has a has_voted flag that is set to 1 after their vote is recorded. Any second attempt triggers a fraud lockdown that requires admin intervention.

Admin password protection
   The admin password is required at login, to end voting early, and to unlock after a fraud alert. Three wrong attempts at login locks the system for 10 seconds.

Face verification at entry and exit
   Two biometric checks per voter. One to enter the booth and one to exit. Failing either triggers an alert.

Rate limiting
   A maximum of 3 votes per 60-second window prevents booth flooding. Exceeding the limit pauses the system for 10 seconds.

Audit logging
   Every important event is logged with a timestamp to /tmp/vote_audit.log including VOTE_CAST, DUPLICATE_VOTE, FACE_FAIL, FRAUD_LOCKDOWN, UNKNOWN_VOTER, and EXIT_FAIL_LOCKED.

Priority-inherit mutexes
   All 5 mutexes use PTHREAD_PRIO_INHERIT to prevent priority inversion, which is a common bug in real-time systems where a low-priority thread holding a lock blocks a high-priority thread indefinitely.


QNX CONCEPTS USED

QNX IPC Message Passing
   Votes are not counted directly in the main thread. Instead the main thread sends a message to the counter thread using MsgSend and blocks until the counter thread validates the vote and replies using MsgReply. This makes vote counting atomic and auditable. This is the standard QNX way for processes and threads to communicate safely.

SCHED_FIFO Real-Time Scheduling
   All threads use SCHED_FIFO scheduling. Once a thread starts running it keeps running until it blocks or a higher-priority thread needs the CPU. There is no time-slicing interruption.

Thread Priorities
   counter_task runs at priority 25 (highest) - vote counting is never delayed
   timer_task runs at priority 10 - updates the countdown every second
   main thread runs at priority 5 (lowest) - handles console input

ChannelCreate and ConnectAttach
   ChannelCreate creates a message channel that the counter thread listens on.
   ConnectAttach creates a connection to that channel so the main thread can send messages to it.

Priority Inheritance Mutexes
   Prevents priority inversion by temporarily boosting the priority of a low-priority thread that holds a lock needed by a high-priority thread.


SYSTEM ARCHITECTURE

   main thread (priority 5)
      handles all console input and output
      sends vote messages to counter_task via MsgSend
      blocks and waits for MsgReply before continuing

   counter_task thread (priority 25)
      sits on MsgReceive waiting for vote messages
      validates voter index, duplicate check, candidate index
      updates voter_db and candidates arrays
      sends reply back via MsgReply

   timer_task thread (priority 10)
      counts down from 1200 seconds
      sets voting_open to 0 when time expires
      closes the voting session automatically


REGISTERED VOTERS

17 voters are pre-registered with IDs 101 to 117.
Names: Arsh, Kratika, Anushree, Diana, Gohar, Kvsss, Ravindra, Chandra, Alish, Kumar, Anupama, Dim, Rahul, Virat, Siddaramayiya, Kumarswami, Ravikiran


TECH STACK

Language        C (C99)
OS              QNX Neutrino RTOS
Threading       POSIX threads with SCHED_FIFO real-time scheduling
IPC             QNX native message passing via sys/neutrino.h
Synchronization 5 priority-inherit mutexes



