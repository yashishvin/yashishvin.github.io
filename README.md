TEAM MEMBERS
1. Thanuja Mudududla
2. Yashishvin Pothuri (018222113)

Q1. Task Division
  Environment Setup
- Thanuja: Set up the outer VM with working nested virtualization. Installed prerequisites, dependencies, and cloned the Linux kernel from the official GitHub repository.
- Yash: Controlled cloud VM settings, configured remote desktop, and validated nested virtualization capabilities.

  Kernel Build:
Yash: Configured kernel build settings, fixed build problems, and at the installation in the outer VM of the prepared kernel. Created snapshots of the VM as a safety procedural before installation.
Thanuja: Aided in build configuration and verified the integrity of the environment post-kernel installation.

  KVM Code Modifications:
Yash: Changed the KVM exit handler in the kernel source code to add counters in the exit types and incorporated printk logging every 10,000 exits.
Thanuja: Performed code change review for correctness and consistency by suggesting improvements on needed items.

  Testing:
Yash: Recompiled the kernel with KVM changes and validated with tests that the modified kernel in the outer VM was functional.
Thanuja: Booted the inner VM with the changed kernel, checked dmesg logs, and validated that exit counts and logs were correct.

  Documentation and Analysis:
Thanuja: Documented the KVM code changes, noting which parts of code were changed and how the exit-counting mechanism was implemented.
Yash: Documented the procedures for testing, including the various workloads used to check exit types and the findings observed from test work.

  Q2. Documentation -

   Step 1: Environment Setup - Setting up Google Cloud VM
To begin the assignment, we re-created the setup from Assignment 1 to establish a solid foundation for the tasks in Assignment 2. This involved creating the outer VM on Google Cloud, ensuring that it was configured with nested virtualization capabilities to support the requirements of this assignment. We also installed and configured essential tools, such as Google Chrome
Remote Desktop, allowing us to easily access and manage the VM remotely for further development and testing. Additionally, we set up the inner VM within the outer environment, replicating the nested virtualization setup used in Assignment 1. This inner VM provided the environment necessary to thoroughly test the changes we planned to make to the kernel's KVM module, facilitating accurate and isolated evaluation of our modifications. This initial setup ensured a seamless transition into the new tasks, allowing us to focus on kernel development and testing without setup complications. 

   Step 2:
Following the initial setup, we moved on to the kernel build phase, where we prepared the environment for modifying the KVM hypervisor. First, we forked the Linux kernel source code from the main repository on GitHub into our personal GitHub accounts.
This provided us with a dedicated space to make and track changes.

We then cloned our forked repository into the outer VM, which served as our main development environment.
git clone https://github.com/Thanuja0911/linux.git
cd linux

Then we install the necessary packages to build a Linux kernel using below commands:

sudo apt update
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev

Then, using the cloned kernel source, we configured the kernel build environment. This included using make menuconfig to ensure the necessary components, like KVM support, were enabled for our custom kernel build.
We also need to ensure that the following lines in your config file are set to empty strings if not you may come across compilation errors related to certificates, you can edit your file using text editors like vim or nano

CONFIG_SYSTEM_REVOCATION_KEYS=""
CONFIG_SYSTEM_TRUSTED_KEYS =""

We proceeded with compiling the kernel with command: make -j$(nproc). The compilation takes a while it depends on the size of your RAM and the number of cores on your system, if your build is successful you should see a message saying that the Kernel is ready. After the build execute the command make modules
Before installing the custom-built kernel, we took a snapshot of the outer VM to safeguard our progress, in case any issues arose with the new kernel. You can take a snapshot from the google cloud console in the Compute Engine dashboard navigate to Storage>Snapshots>CreateSnapShot

Once the build was successful, we installed the new kernel through the following commands

sudo make modules_install (Installs the compiled kernel modules)

sudo make install(installs the compiled linux and related files( like the kernel image and System.map)

If the grub is not updated with the previous command we can use the following command

sudo update-grub

After this we rebooted the outer VM using the command
sudo reboot

After rebooting We verified that the new kernel works by checking the version of the kernel with the command 
uname -r

After completing these steps we will have the kernel installed. This setup allowed us to begin modifying the kernel code with a stable and verified environment, ready to implement the KVM-specific changes.
We then modified the KVM source code to implement the assignment requirements. We began by locating the KVM exit handling code within the kernel's source files. Specifically, this involved identifying the __vmx_handle_exit function located in(~/linux/arch/x86/kvm/vmx) the KVM module, where VM exits are handled. 
Here, we defined our global total exit counter and a global counter for each exit, for every 10,000 exits we print the stats for how many exits we encountered and how many time we encountered them we do this through the print k function, we print a human-readable name of the exit along with the exit code and the number of times the exit occurred. You can find a snippet of the code below( get_exit_reason_name converts the exit reason to a human readable name

THE CODE SNIPPET WE ADDED INTO VMX.C
static unsigned long long exit_counters[256] = {0}; // per-exit-type counters
static unsigned long long total_exit_count = 0; // total exit counter

/* Helper function which map exit codes to human readable names */
static const char *get_exit_reason_name(int reason) {
    switch (reason) {
    case 0:  return "EXCEPTION_NMI";                  // Non-maskable interrupt or exception
    case 1:  return "EXTERNAL_INTERRUPT";             // External interrupt
    case 2:  return "TRIPLE_FAULT";                   // Triple fault
    case 3:  return "INIT_SIGNAL";                    // INIT signal
    case 4:  return "STARTUP_IPI";                    // Startup IPI
    case 5:  return "IO_SMI";                         // SMI during I/O instruction
    case 6:  return "OTHER_SMI";                      // SMI other than I/O
    case 7:  return "INTERRUPT_WINDOW";               // Interrupt window exiting
    case 8:  return "NMI_WINDOW";                     // NMI window exiting
    case 9:  return "CPUID";                          // CPUID instruction
    case 10: return "GETSEC";                         // GETSEC instruction
    case 11: return "HLT";                            // HLT instruction
    case 12: return "INVD";                           // INVD instruction
    case 13: return "INVLPG";                         // INVLPG instruction
    case 14: return "RDPMC";                          // RDPMC instruction
    case 15: return "RDTSC";                          // RDTSC instruction
    case 16: return "RSM";                            // RSM instruction in SMM
    case 17: return "VMCALL";                         // VMCALL instruction
    case 18: return "VMCLEAR";                        // VMCLEAR instruction
    case 19: return "VMLAUNCH";                       // VMLAUNCH instruction
    case 20: return "VMPTRLD";                        // VMPTRLD instruction
    case 21: return "VMPTRST";                        // VMPTRST instruction
    case 22: return "VMREAD";                         // VMREAD instruction
    case 23: return "VMRESUME";                       // VMRESUME instruction
    case 24: return "VMWRITE";                        // VMWRITE instruction
    case 25: return "VMXOFF";                         // VMXOFF instruction
    case 26: return "VMXON";                          // VMXON instruction
    case 27: return "CR_ACCESS";                      // Control register access
    case 28: return "DR_ACCESS";                      // Debug register access
    case 29: return "IO_INSTRUCTION";                 // I/O instruction
     case 30: return "MSR_READ";                       // MSR read
    case 31: return "MSR_WRITE";                      // MSR write
    case 32: return "INVALID_GUEST_STATE";            // Invalid guest state
    case 33: return "EPT_VIOLATION";                  // EPT violation
    case 34: return "EPT_MISCONFIG";                  // EPT misconfiguration
    case 35: return "INVEPT";                         // INVEPT instruction
    case 36: return "RDTSCP";                         // RDTSCP instruction
    case 37: return "VMX_PREEMPTION_TIMER_EXPIRED";   // VMX preemption timer expired
    case 38: return "INVVPID";                        // INVVPID instruction
    case 39: return "WBINVD";                         // WBINVD instruction
    case 40: return "XSETBV";                         // XSETBV instruction
    case 41: return "APIC_WRITE";                     // APIC write
    case 42: return "RDRAND";                         // RDRAND instruction
    case 43: return "INVPCID";                        // INVPCID instruction
    case 44: return "VMFUNC";                         // VMFUNC instruction
    case 45: return "ENCLS";                          // ENCLS instruction
    case 46: return "RDSEED";                         // RDSEED instruction
    case 47: return "PAGE_MODIFICATION_LOG_FULL";     // PML buffer full
    case 48: return "XSAVES";                         // XSAVES instruction
    case 49: return "XRSTORS";                        // XRSTORS instruction
    default: return "UNKNOWN_EXIT";                   // Catch-all for unrecognized exit codes
    }
}

The function log_exit_counts prints the statistics for the number of exits

static void log_exit_counts(void) {
    printk(KERN_INFO "KVM Exit Stats (Every 10,000 Exits):\n");
    for (int exit_code = 0; exit_code < 256; exit_code++) {
        if (exit_counters[exit_code] > 0) {
            printk(KERN_INFO "  Exit Code: %d (%s) - %llu occurrences\n",
                   exit_code, get_exit_reason_name(exit_code), exit_counters[exit_code]);
        }
    }
}

/* Code snippet to be added to the KVM exit handler */
        int exit_code = exit_reason.basic; // Extract exit reason
        exit_counters[exit_code]++;        // Increment per-exit-type counter
        total_exit_count++;                // Increment global counter

        /* Log every 10,000 exits */
        if (total_exit_count % 10000 == 0) {
          log_exit_counts();
          printk(KERN_INFO "Total VM Exits so far: %llu\n", total_exit_count );
        }

Add the above code snippet towards the start of the function __vmx_handle_exit
After we have made the changes save it and rebuild and install the kernel using the following commands.

1)make -j$(nproc)( Build the kernel)
2)make modules(Build the modules)
3)sudo make modules_install (Installs the compiled kernel modules)
4)sudo make install(installs the compiled linux and related files( like the kernel image and System.map)

Once this is done reboot the system using the command
Sudo reboot

After logging back in to the outer VM To study the behaviour of exits we need to restart our inner vm we did it through the virt manager GUI on the outer VM, which we accessed through google remote desktop connection.

To check the type of exits occurring we ran the following command on the outer VM

sudo dmesg | grep "  Exit Code"

This printed the statistic of the different types of exits and the number of times they were occurring.

Q3)Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there
 							
more exits performed during certain VM operations? Approximately how many exits does a full VM
 							
boot entail? 


Ans) We found that the number of exits does not exit at a stable rate it depends on the operations being performed, operations which require frequent interaction with the hypervisor. We found that operations like booting the VM or installing OS entails a higher number of exits. For us a VM boot took about 600,000 exits.

Q4)Of the exits which are the most frequent and least

Ans) From the stats we observed that there were high frequency of MSR_READ exits
Some of the least frequent exits we saw were IO_INSTRUCTION
