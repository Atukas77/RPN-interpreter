	# IO interrupts
	.data
kb_buf_head:	.word  	0			# pointer to most recent data
kb_buf:		.space 	100			# 25 words / 100 bytes
kb_buf_tail:	.word	0			# pointer to most recently processed data
comma:		.asciiz ", "			#
newline:	.asciiz "\n"			#


################################################################################################
	# exception handling / IRQs
	.ktext 	0x80000180
	
	mfc0	$k0, $13			# move cause contents to k0
	srl	$k0, $k0, 2			# shift right by 2
	andi	$k0, $k0, 0x1F			# mask o7ff final 5 bits
	beq	$k0, $zero, IRQ			# 0 -> hardware interrupt
	
	
	eret					# return to normal code

IRQ:    # check where interrupt came from:
	mfc0	$k0, $13			# move cause contents to k0
	andi	$k1, $k0, 0x0100		# 0x0100 = 8 = keyboard
	bne	$k1, $zero, IRQ_KB		# jump to kb IRQ
	
	# add other IRQ/devices bit checks here
	eret					# return to normal code

IRQ_KB:
	lw	$k1, kb_buf_head		# pull current size of buffer
	
	li	$k0, 0xffff0004			# keyboard data register
	lw	$k0, 0($k0)			# get character
	sw	$k0, 0($k1)			# store in buffer
	add	$k1, $k1, 4			# increment to cover new character
	
	# if head > buf + 100(bytes): reset head -> 0
	la	$k0, kb_buf			# get buffer address
	addi	$k0, $k0, 100			# offset by length
	beq	$k0, $k1, reset_head		# head == buf + 100?
	
	sw	$k1, kb_buf_head		# push new head pointer back
	eret					# all done, return 

reset_head:
	addi	$k0, $k0, -100			# undo offset
	sw	$k0, kb_buf_head 		# reset head pointer to start of buffer
	eret					# all done, return


################################################################################################
	# section for processing pressed keys 
	.text
	j 	main
	
	
new_key:
	# check if new key available to process
	# return 0 if yes, return 1 if no
	lw	$t0, kb_buf_head
	lw	$t1, kb_buf_tail
	seq	$v0, $t0, $t1
	
	
	jr	$ra
	

process_key:
	# if new key has been pressed, process it
	addi	$gp, $gp, -4			# make space on heap (we use the heap so it doesn't interfere with stack
	sw	$ra, 0($gp)			# store ra ahead of nested call

	jal	new_key				# check if new key pressed available
	bnez	$v0, exit_all			# if not, exit
	
	# new key available to process
	
	lw	$t0, kb_buf_tail 		# get tail pointer
	lw	$t1, ($t0)			# dereference to get key
	
	
	blt	$t1, 48, continue_process	# if pressed key is not a digit 1-9, 
	bgt	$t1, 57, continue_process	# we don't put it on the stack
			
	addi	$sp, $sp, -4			# make space on stack
	addi	$t1, $t1, -48      		# convert ascii value to digit
	sw	$t1, ($sp)			# save digit on stack
	addi	$s4, $s4, 4			# keep track of local stack size
continue_process:
	beq	$t1, ' ', process_int		# turn individual digits into decimal integer when spacebar pressed
	beq	$t1, '+', addition
	beq	$t1, '-', subtraction
	beq	$t1, '/', division
	beq	$t1, '*', star_counter
	beq	$t1, ';', s_counter
	beq	$t1, '\n', terminate		# exit program if 'enter' key is pressed
	# if we reach here, we ignore the key press	
	j	exit_moveptr


##################################################################################
# section for processing key values saved on the stack
	

process_int:
	
			
	beq	$s1, 1, multiplication		# if star counter == 1, goto multiplication
	beq	$s1, 2, power			# else if star counter == 2, goto power
	
	beq	$s4, $zero, continue		# skip processing integer if spacebar pressed more than once (local stack size is zero)
	
	li	$t2, 1				# multiplier
	li	$t4, 0				# counter
	li	$t5, 0				# temp
	li	$t6, 0				# register to store converted integer
process_loop: 
	# looping through every digit to convert to decimal number
	bge	$t4, $s4, end_p_loop		# loop until counter reaches stack size
	lw	$t5, ($sp)			# store current item on stack in temp register
	mul	$t5, $t5, $t2			# multiply temp by multiplier
	add	$t6, $t6, $t5			# add temp to result register
	addi	$sp, $sp, 4			# increment stack pointer
	mul	$t2, $t2, 10			# increase multiplier 10 fold
	addi	$t4, $t4, 4			# increment counter
	j	process_loop
end_p_loop:
	li	$s4, 0				# set stack size to 0 since loop freed the stack
	
	addi	$sp, $sp, -4			# make space on stack again
	sw	$t6, ($sp)			# store converted number back on stack
	addi	$s3, $s3, 4			# keep track of global stack size
	
continue:			
	
	lw	$a0, 4($sp)			# print last two items on stack to console for debugging
	li	$v0, 1
	syscall
	la	$a0, comma
	li	$v0, 4
	syscall
	lw	$a0, ($sp)
	li	$v0, 1
	syscall
	la	$a0, newline
	li	$v0, 4
	syscall

###############################################################################
	# printing to display
	
	beq	$s2, $zero, exit_moveptr	# if semicolon wasn't preesed (s_counter==0), don't print
	li	$s2, 0				# reset semicolon counter
	beq	$s3, $zero, exit_moveptr	# if stack is empty, don't print
				
	lw	$a0, ($sp)			# load top of stack
	beq	$a0, $zero, exit_moveptr	# if top of stack equal to 0, don't print
	lui 	$t5, 0xffff			# load IO address
	lw  	$t4, 8($t5)  			# load ready bit address
	andi    $t4, $t4, 1			# ready bit
	sw  	$a0, 0xc($t5)  			# store item to print
	
	
						
	add	$sp, $sp, $s3			# empty stack 

	j	exit_moveptr
	

##################################################################################
exit_moveptr:
	# update tail pointer then forward to final exit
	addi	$t1, $t0, 4			# move pointer forward
	# check if we need to wrap tail pointer 
	la	$t0, kb_buf			# get buffer address
	addi	$t0, $t0, 100			# offset by length
	beq	$t0, $t1, reset_tail		# head == buf + 100?
	
	sw	$t1, kb_buf_tail		# push new tail pointer back
	j	exit_all

reset_tail:
	# wrap tail pointer round to start of queue
	addi	$t0, $t0, -100			# undo offset
	sw	$t0, kb_buf_tail		# push new pointer to RAM
	j	exit_all


exit_all:
	lw	$ra, 0($gp)			# restore return address
	addi	$gp, $gp, 4			# restore global pointer
	jr	$ra				# exit
	
##################################################################################
# calculation functions
# pop last two operands from stack, perform calculation, push result back on stack

addition: 	# called when '+' is pressed
	lw	$t2, 4($sp)			# load first operand from stack
	lw	$t3, ($sp)			# load second operand
	add	$t2, $t2, $t3			# perform addition
	addi	$sp, $sp, 4			# free last element of stack
	sw	$t2, ($sp)			# save result back on stack
	addi	$s3, $s3, -4			# decrease stack size by 4 bytes
	li	$t2, 0				# reset operand registers
	li	$t3, 0		
	j	exit_moveptr

		
subtraction: 	# called when '-' is pressed
	lw	$t2, 4($sp)			# load first operand from stack
	lw	$t3, ($sp)			# load second operand
	sub	$t2, $t2, $t3			# perform subtraction
	addi	$sp, $sp, 4			# free last element of stack
	sw	$t2, ($sp)			# save result back on stack
	addi	$s3, $s3, -4			# decrease stack size by 4 bytes
	li	$t2, 0				# reset operand registers
	li	$t3, 0	
	j	exit_moveptr


division: 	# called when '/' is pressed, integer division only
	lw	$t2, 4($sp)			# load first operand from stack
	lw	$t3, ($sp)			# load second operand
	beq	$t3, $zero, exit_moveptr	# if denominator is zero, skip instruction
	div	$t2, $t2, $t3			# perform division
	addi	$sp, $sp, 4			# free last element of stack
	sw	$t2, ($sp)			# save result back on stack
	addi	$s3, $s3, -4			# decrease stack size by 4 bytes
	li	$t2, 0				# reset operand registers
	li	$t3, 0		
	j	exit_moveptr


star_counter:	# count how many times '*' pressed to differentiate between mult and power
	addi	$s1, $s1, 1
	j	exit_moveptr
	
	
s_counter:	# keep track if semicolon has been pressed
	addi	$s2, $s2, 1
	j	process_int
	
	
multiplication:	# called when '*' is pressed once, followed by spacebar or semicolon
		# no overflow
	li	$s1, 0				# reset '*' key counter	
	lw	$t2, 4($sp)			# load first operand from stack
	lw	$t3, ($sp)			# load second operand
	mul	$t2, $t2, $t3			# perform multiplication
	addi	$sp, $sp, 4			# free last element of stack
	sw	$t2, ($sp)			# save result back on stack
	addi	$s3, $s3, -4			# decrease stack size by 4 bytes
	li	$t2, 0				# reset operand registers
	li	$t3, 0				#
	j	continue			# go back to int processing part
	
	
power:	# called when '*' is pressed twice, followed by spacebar or semicolon
	li	$s1, 0				# reset '*' counter
	li	$t2, 0				# loop counter
	li	$t4, 1				# result
	lw	$t7, 4($sp)			# load first operand from stack (base)
	lw	$t8, ($sp)			# load second operand (exponent)
power_loop:	
	bge	$t2, $t8, end_power_loop	# if counter > exponent then end loop
	mul	$t4, $t4, $t7			# result *= base
	addi	$t2, $t2, 1			# counter++
	j	power_loop
end_power_loop:
	addi	$sp, $sp, 4			# free last element of stack
	sw	$t4, ($sp)			# save result back on stack
	addi	$s3, $s3, -4			# decrease stack size by 4 bytes
	li	$t7, 0				# reset operand registers
	li	$t8, 0				#
	j	continue			# go back to int processing part


##################################################################################
main:
	# set up queue
	la	$t0, kb_buf			# get index 0 of buffer
	sw	$t0, kb_buf_head		# current head = 0
	sw	$t0, kb_buf_tail		# current tail = 0
	
	li	$t0, 0xffff0000			# keyboard control register
	li	$t1, 2				# interrupt enable bit -> 1
	sw	$t1, ($t0)			# enable interrupts on keyboard

loop:
	addi	$s0, $s0, 1			# key track of how many times we have done this
	jal	process_key			# check for and process key presses
	j 	loop				# loop forever
	
##################################################################################
	
terminate:	# come here when 'enter' key is pressed to exit program
	li	$v0, 10
	syscall
	
	
	
	
	
	
	
	
	
	
