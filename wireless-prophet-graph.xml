# and only forwards those which it hasn't seen before.

# Each message is of the form "ID:DATA" where ID is some arbitrary
# message identifier and DATA is the payload.  In order to reduce
# memory usage, the agent stores only the message ID.

# Note that I have not put in any mechanism to expire old message IDs
# from the list of seen messages.  There also isn't any mechanism
# for assigning message IDs.


Mac/Simple set bandwidth_ 1Mb

set MESSAGE_PORT 42
set BROADCAST_ADDR -1


# variables which control the number of nodes and how they're grouped
# (see topology creation code below)
#set group_size 4
#set num_groups 6

set sim_stop 0

set num_nodes 8
set grid_x 1600
set grid_y 1600



set val(chan)           Channel/WirelessChannel    ;#Channel Type
set val(prop)           Propagation/TwoRayGround   ;# radio-propagation model
set val(netif)          Phy/WirelessPhy            ;# network interface type



#set val(mac)            Mac/802_11                 ;# MAC type
#set val(mac)            Mac                 ;# MAC type
set val(mac)		Mac/Simple


set val(ifq)            Queue/DropTail/PriQueue    ;# interface queue type
set val(ll)             LL                         ;# link layer type
set val(ant)            Antenna/OmniAntenna        ;# antenna model
set val(ifqlen)         50                         ;# max packet in ifq


# DumbAgent, AODV, and DSDV work.  DSR is broken
set val(rp) DumbAgent
#set val(rp)             DSDV
#set val(rp)             DSR
#set val(rp)		 AODV



#threshold per le delivery_pred
set threshold 0.10

#prophet constant
set Pinit 0.75
set Beta  0.25
set Gamma 0.98
set k 2

#prophet zone (ordine orario a partire da 0.0)
set z0_x 1
set z0_y 1

set z1_x 1
set z1_y 799

set z2_x 799
set z2_y 799

set z3_x 799
set z3_y 1


for {set i 0} {$i<4} {incr i} {
	set zone_list($i) null
}

#memorie di au filli
#remove by value
#set idx [lsearch $mylist "b"]
#set mylist [lreplace $mylist $idx $idx]


# size of the topography
set val(x)  $grid_x
set val(y)  $grid_y


set ns [new Simulator]
set f [open wireless-prophet-f.tr w]
set ftracer [open wireless-prophet-tracer.tr w]
set fspyer [open wireless-prophet-spyer.tr w]
set fgraphSpy [open wireless-prophet-graph_spy.xml w]


$ns trace-all $f


set nf [open wireless-prophet-$val(rp).nam w]
$ns namtrace-all-wireless $nf $val(x) $val(y)



# write on file tracer
proc writeFT { data } {
	global ftracer
	puts -nonewline $ftracer $data
}


#write on file spyer
proc writeFS { data } {
	global fspyer
	puts -nonewline $fspyer $data

}

writeFT "File trace\n "
writeFS "File spy\n "
puts  -nonewline $fgraphSpy "<xml version=\"1.0\" encoding=\"UTF-8\">\n"



# set up topography object
set topo       [new Topography]

$topo load_flatgrid $val(x) $val(y)

#
# Create God
#

create-god $num_nodes



set chan_1_ [new $val(chan)]

$ns node-config -adhocRouting $val(rp) \
                -llType $val(ll) \
                -macType $val(mac) \
                -ifqType $val(ifq) \
                -ifqLen $val(ifqlen) \
                -antType $val(ant) \
                -propType $val(prop) \
                -phyType $val(netif) \
                -topoInstance $topo \
                -agentTrace OFF \
                -routerTrace OFF \
                -macTrace OFF \
                -movementTrace OFF \
                -channel $chan_1_ 


# subclass Agent/MessagePassing to make it do Prophet

Class Agent/MessagePassing/Prophet -superclass Agent/MessagePassing


proc tcl::mathfunc::roundto {value decimalplaces} {expr {round(10**$decimalplaces*$value)/10.0**$decimalplaces}} 

# richiamata dalla receive ogni volta che il nodo a incontra il nodo b
Agent/MessagePassing/Prophet instproc meetNodeProb {nodea nodeb } {
	global delivery_pred Pinit ns k a
	

	set delivery_pred($nodea,$nodeb) [format "%.4f" [expr ($delivery_pred($nodea,$nodeb) + ((1 - $delivery_pred($nodea,$nodeb) )) * $Pinit)]]
	set current_time [format "%.2f" [expr {double(round(100*[$ns now]))/100}]]
	set delivery_pred($nodea,$nodeb,"time") $current_time	
	
	
	writeFT "\n$delivery_pred($nodea,$nodeb,"time") -meetNodeProb- => prob($nodea,$nodeb):$delivery_pred($nodea,$nodeb)" 
	

}


#richimata dallo scheduler ogni tot secondi 
Agent/MessagePassing/Prophet instproc agingNodeProb {nodea nodeb } {


	global ns k Gamma delivery_pred sim_stop

	set current_time [format "%.2f" [expr {double(round(100*[$ns now]))/100}]]
	
	set delivery_pred($nodea,$nodeb) [format "%.4f" [expr ($delivery_pred($nodea,$nodeb) * $Gamma**$k)]]	
	set delivery_pred($nodea,$nodeb,"time") $current_time	
		
 	writeFT "\n$delivery_pred($nodea,$nodeb,"time") -agingNodeProb- => prob($nodea,$nodeb):$delivery_pred($nodea,$nodeb)"
	
	
	
}


#richimata dalla receive quando leggo nella delivery pred del node sender 
Agent/MessagePassing/Prophet instproc transitiveNodeProb {nodea nodeb nodec} {
	global ns k Gamma Beta delivery_pred a

	set current_time [format "%.2f" [expr {double(round(100*[$ns now]))/100}]]

	set delivery_pred($nodea,$nodec) [format "%.4f" [expr ($delivery_pred($nodea,$nodec) + (1 - $delivery_pred($nodea,$nodec)) * $delivery_pred($nodea,$nodeb) * $delivery_pred($nodeb,$nodec) * $Beta )]]

	set delivery_pred($nodea,$nodec,"time") $current_time	

	# every k unit time aging function is called 
	#$ns after $k "$a($nodea) agingNodeProb $nodea $nodec"
	#puts "\nac:$delivery_pred($nodea,$nodec) -- ab:$delivery_pred($nodea,$nodeb) -- bc:$delivery_pred($nodeb,$nodec)"
	writeFT "\n$delivery_pred($nodea,$nodec,"time") -transitiveNodeProb- => prob($nodea,$nodec):$delivery_pred($nodea,$nodec)"
}


Agent/MessagePassing/Prophet instproc agingNodeProbManager {nodea} {
	global ns k delivery_pred a num_nodes sim_stop
	$self instvar node_

	for {set i 0} {$i<$num_nodes} {incr i} {
		
		if {( $i != $nodea ) && ( $delivery_pred($nodea,$i) != 0 )} {
			
			$a($nodea) agingNodeProb $nodea $i

		}


	}

	
	# every k unit time aging function is called 
	if {$sim_stop == 0} {
		$ns after $k "$a($nodea) agingNodeProbManager $nodea"
	}


}



###		      ###
###  RECEIVE FUNCTION ###
###		      ###
# qui dentro devo fare i controlli prophet
Agent/MessagePassing/Prophet instproc recv {source sport size data} {
    $self instvar messages_seen node_ 
    global ns BROADCAST_ADDR  delivery_pred friends a num_nodes array_spy fspyer threshold fgraph fgraphSpy



# extract message ID from message
    set packet_id [lindex [split $data ":"] 0]
    set packet_data [lindex [split $data ":"] 1]
    

	# $source=packet sender       $[$node_ node-addr]=packet receiver
	set packet_receiver [$node_ node-addr]
    

	set current_time [format "%.2f" [expr {double(round(100*[$ns now]))/100}]]
	
	if {$a($packet_receiver,'type') == 0} { #node type 0=normal node
	
		writeFT "\n$current_time -recvPacket- => pkt_id:$packet_id (NormalNode) pkt_source:$source pkt_receiver:$packet_receiver"
    	

		### DELIVERY PRED ALG START ###

		#start aging
		if {$delivery_pred($packet_receiver,"flag_aging") != 1} {
			set delivery_pred($packet_receiver,"flag_aging") 1
			$a($packet_receiver) agingNodeProbManager $packet_receiver
		}

		#set probability of meet of node that send packet ( i meet it in this area )
		$a($packet_receiver) meetNodeProb $packet_receiver $source
		# add meet node in friends list
		if { [lsearch $friends($packet_receiver) $source] == -1 } { 
			if {$friends($packet_receiver) == "null"} {
				set friends($packet_receiver) $source
			} else {
				lappend friends($packet_receiver) $source  
			}  
			writeFT "\n$current_time -addNewFriend- => node:$packet_receiver friends:$friends($packet_receiver)"
		}
	

		#set probability of every node that aren't receiver friend ( not set if is pkt_sender or is already receiver friend or x is me)
		for {set x 0} {$x < $num_nodes} {incr x} {
			if { ([lsearch $friends($packet_receiver) $x] == -1) && ( $x != $source) && ( $x != $packet_receiver ) && ( $delivery_pred($source,$x) != 0)} {

				#set transitive receiver source x
				$a($packet_receiver) transitiveNodeProb $packet_receiver $source $x
			
			} 


		}


		### DELIVERY PRED ALG FINISH ###




	


		if {[lsearch $messages_seen $packet_id] == -1} {
			lappend messages_seen $packet_id  
		} 


	} else { #node type 1=spy node
		
		
		writeFT "\n$current_time -recvPacket- => pkt_id:$packet_id (SpyNode) pkt_source:$source pkt_receiver:$packet_receiver"
		
		for {set i 0} {$i<$num_nodes} {incr i} {
			if {$array_spy($source) == "null"} {
				set array_spy($source) $delivery_pred($source,$i)	
			} else {
				lappend array_spy($source) $delivery_pred($source,$i)
			}
				
			
		}
		writeFS "\n\n$current_time -arraySpy- => delivery_pred_of:$source list:$array_spy($source)"
			set var_source $source
		
			puts  -nonewline $fspyer "\ndelivery_pred of node $source\n"
			puts -nonewline $fgraphSpy "   <nodi>\n     <nodo>$var_source</nodo> \n"
			for {set j 0} {$j<$num_nodes} {incr j} {
				puts -nonewline $fspyer "prob($source,$j)=$delivery_pred($source,$j)  "
			
				if {$delivery_pred($source,$j)>$threshold} {
						
						puts -nonewline $fgraphSpy "  \n                 <amico>$j</amico> \n    "	
				}	
					
					
		
					
					
					
				
			}
			puts -nonewline $fgraphSpy "</nodi>     \n      \n"
		
	}
}





###		      		  		 ###
###  SEND TABLES FUNCTION    ###
###		              		 ###
# la send message sarebbe lo scambio di tabelle quando due nodi si incontrano.
Agent/MessagePassing/Prophet instproc send_tables {size packet_id data port} {
    $self instvar messages_seen node_
    global ns MESSAGE_PORT BROADCAST_ADDR 
    lappend messages_seen $packet_id
	set current_time [format "%.2f" [expr {double(round(100*[$ns now]))/100}]]
	writeFT "\n$current_time -sendPacket- => pkt_id:$packet_id pkt_sender:[$node_ node-addr]"
    
    #$ns trace-annotate "[$node_ node-addr] sending packet $packet_id"
	
    $self sendto $size "$packet_id:$data" $BROADCAST_ADDR $port
}






Agent/MessagePassing/Prophet instproc update_zone {zone} {
	global a zone_list ns
	$self instvar node_
	set current_time [format "%.2f" [expr {double(round(100*[$ns now]))/100}]]
	set old_zone $a([$node_ node-addr],'zone')
	set a([$node_ node-addr],'zone') $zone
	
	writeFT "\n$current_time - aggiornamentoZona => $zone"
	

	set idxNew [lsearch $zone_list($zone) "null"]
	
	set zone_list($zone) [lreplace  $zone_list($zone) $idxNew $idxNew]

	
	
	set idx [lsearch $zone_list($old_zone) "[$node_ node-addr]"]
	
	set zone_list($old_zone) [lreplace  $zone_list($old_zone) $idx $idx]
	
	lappend zone_list($zone) [$node_ node-addr]
	

	for {set i 0} {$i<4} {incr i} {
	
		writeFT "\n$current_time -lista zona$i => $zone_list($i)"
	}	
}




# create a bunch of nodes
for {set i 0} {$i < $num_nodes} {incr i} {
     
	set friends($i) "null" ; # nodes that a node meet in it life
	
	for {set j 0} {$j < $num_nodes} {incr j} { 
		
		set delivery_pred($i,$j) 0 ;   #indica la delivery predicability (indice della tabella) - delivery_pred(nodoacuiappartienelatabella , nodo destinazione)
		set delivery_pred($i,$j,"time") 0.0 ; 
		

	}
	set delivery_pred($i,"flag_aging") 0

    set n($i) [$ns node]

 	set array_spy($i) "null" ; #array dove il nodo spia salva le delivery pred dei nodi che incontra ($i)
   
	
}

#creazione del nodo spia
proc createSpy {zone x y} {
	global num_nodes n ns MESSAGE_PORT z a 
	$n([expr ($num_nodes-1)]) setdest $x $y 8000)
	set current_time [format "%.2f" [expr {double(round(100*[$ns now]))/100}]]
	writeFT "\n$current_time -createspy- => zone_x: $x zone_y: $y"
}



#inizializate nodes position
$n(0) set Y_ $z0_y
$n(0) set X_ $z0_x
$n(0) set Z_ 0.0
$ns initial_node_pos $n(0) 120

$n(1) set Y_ $z0_y
$n(1) set X_ $z0_x
$n(1) set Z_ 0.0
$ns initial_node_pos $n(1) 120

$n(2) set Y_ $z1_y
$n(2) set X_ $z1_x
$n(2) set Z_ 0.0
$ns initial_node_pos $n(2) 120


$n(3) set Y_ $z2_y
$n(3) set X_ $z2_x
$n(3) set Z_ 0.0
$ns initial_node_pos $n(3) 120

$n(4) set Y_ $z2_y
$n(4) set X_ $z2_x
$n(4) set Z_ 0.0
$ns initial_node_pos $n(4) 120

$n(5) set Y_ $z2_y
$n(5) set X_ $z2_x
$n(5) set Z_ 0.0
$ns initial_node_pos $n(5) 120

$n(6) set Y_ $z3_y
$n(6) set X_ $z3_x
$n(6) set Z_ 0.0
$ns initial_node_pos $n(6) 120



# attach a new Agent/MessagePassing/Flooding to each node on port $MESSAGE_PORT
for {set i 0} {$i <[expr ($num_nodes-1)]} {incr i} {
    set a($i) [new Agent/MessagePassing/Prophet]
    $n($i) attach  $a($i) $MESSAGE_PORT
    $a($i) set messages_seen {}
	set a($i,'zone') 0
	set a($i,'type') 0
}

#set spy node 	
$n([expr ($num_nodes-1)]) set Y_ 400.0
$n([expr ($num_nodes-1)]) set X_ 1599.0
$n([expr ($num_nodes-1)]) set Z_ 0.0
$ns initial_node_pos $n([expr ($num_nodes-1)]) 180
set a([expr ($num_nodes-1)]) [new Agent/MessagePassing/Prophet]
$n([expr ($num_nodes-1)]) attach  $a([expr ($num_nodes-1)]) $MESSAGE_PORT
$a([expr ($num_nodes-1)]) set messages_seen {}
set a([expr ($num_nodes-1)],'zone') 3
set a([expr ($num_nodes-1)],'type') 1




# events scheduler

source scenario_tesi1.tcl

proc writeFriends {} {
	global num_nodes friends
	set fgraph [open wireless-prophet-graph.xml w]
	puts  -nonewline $fgraph "<xml version=\"1.0\" encoding=\"UTF-8\">\n"
	
	for {set i 0} {$i<$num_nodes} {incr i} {
		
		puts -nonewline $fgraph "   <nodi>\n     <nodo>$i</nodo> "

		for {set j 0} {$j<[llength $friends($i)]} {incr j} {
			puts -nonewline $fgraph "  \n                 <amico>[lindex $friends($i) $j]</amico> \n    "	
			
		}
	puts -nonewline $fgraph "</nodi>     \n      \n"
	}
	
	close $fgraph
}

proc writeDeliv {} {
	global num_nodes delivery_pred friends 
	set fdeliv [open wireless-prophet-deliv.tr w]	
	for {set i 0} {$i<$num_nodes} {incr i} {
		puts  -nonewline $fdeliv "\nfriends of node $i: $friends($i)\n"	
		puts  -nonewline $fdeliv "delivery_pred of node $i\n"
		for {set j 0} {$j<$num_nodes} {incr j} {
			puts -nonewline $fdeliv "prob($i,$j)=$delivery_pred($i,$j)\n"
			
		}
		puts -nonewline $fdeliv "\n---------------------\n"
	}
	close $fdeliv
}
proc finish {} {
        global ns f val ftracer fspyer nf 
		writeDeliv 
		writeFriends
        flush $ftracer
	flush $fspyer
	close $ftracer 
	close $fspyer 
	close $nf
	

        

       	puts "running nam..."
       	#exec nam wireless-prophet-$val(rp).nam &
		
        exit 0
}

$ns run

