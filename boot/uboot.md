env:

#define CFG_NO_FLASH            1
#define CFG_ENV_IS_IN_NAND      1
#define CFG_ENV_SIZE            0x40000 /* 128k Total Size of Environment Sector
#define CFG_ENV_OFFSET_OOB      1       /* Location of ENV stored in block 0 OOB
#define CFG_PREBOOT_OVERRIDE    1       /* allow preboot from memory */
#define NAND_MAX_CHIPS          1
#define CFG_NAND_BASE           0x4e000000
#define CFG_MAX_NAND_DEVICE     1

CFG_ENV_OFFSET is defined to be env_offset, which is set to CFG_OFFSET_OOB

all starts at lib_arm:start_armboot

env_init set directly gd->env_addr to default_environment and gd->env_valid = 1

after nand_init, env_relocate is called, which
	env_ptr is set to malloc(some size);
	env_get_char = env_get_char_memory;
	if (preboot_override) gd->env_valid = 0;
	if (gd->env_valid) env_relocate_spec() else default_env();
	gd->env_addr = env_ptr->data;
	
"Found Environment offset in OOB.." -> try to read env from nand
if not "*** Warning - bad CRC or NAND, using default environment\n\n" -> env from nand is used

main_loop:

if preboot_override, run_command(p);
bootdelay set, s = bootcmd, then
if (!nobootdelay && bootdelay >= 0 && s && !abortboot (bootdelay)) run_command(s);
	abortboot return true when any key pressed

`nand' command:
many nand subcommands indirectly calls mtdparts_init(), which gets parts info from getenv("mtdparts") (and others)

`dynenv' command:
it indirectly calls into mtdparts_init too.  On return, CFG_ENV_OFFSET is set to the new address

`saveenv' save env to CFG_ENV_OFFSET
