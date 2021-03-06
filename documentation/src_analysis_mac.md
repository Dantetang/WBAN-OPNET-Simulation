# MAC 层源码分析

下图为 MAC 层模块的有限状态机实现，看似简单，其实是由于协议逻辑比较复杂，所以将大部分状态跳转都放进了源码里头，并没有直接体现在有限状态机转化图上。

![FSM-MAC](./images/fsm_mac.png)

如上图所示，MAC 层模块主要由三部分状态机实现，绿色的`init`模块为初始化过程，`wait_beacon`为等待beacon帧的状态机，最后的`idle`为核心状态跳转，主要在程序内部使用自中断和中断处理函数组成。

## init 初始化

`init`模块的入口代码为：

```c
wban_mac_init ();

//op_intrpt_priority_set (OPC_INTRPT_SELF, START_OF_GTS_PERIOD, -4);
op_intrpt_priority_set (OPC_INTRPT_SELF, END_OF_EAP1_PERIOD_CODE, -1);
op_intrpt_priority_set (OPC_INTRPT_SELF, START_OF_RAP1_PERIOD_CODE, -2);
op_intrpt_priority_set (OPC_INTRPT_SELF, END_OF_RAP1_PERIOD_CODE, -1);
op_intrpt_priority_set (OPC_INTRPT_SELF, START_OF_MAP1_PERIOD_CODE, -2);
op_intrpt_priority_set (OPC_INTRPT_SELF, END_OF_MAP1_PERIOD_CODE, -1);
op_intrpt_priority_set (OPC_INTRPT_SELF, SEND_B2_FRAME, -2);
op_intrpt_priority_set (OPC_INTRPT_SELF, START_OF_MAP2_PERIOD_CODE, -2);
op_intrpt_priority_set (OPC_INTRPT_SELF, END_OF_MAP2_PERIOD_CODE, -1);
op_intrpt_priority_set (OPC_INTRPT_SELF, START_OF_CAP_PERIOD_CODE, -2);
op_intrpt_priority_set (OPC_INTRPT_SELF, END_OF_CAP_PERIOD_CODE, -1);
op_intrpt_priority_set (OPC_INTRPT_SELF, BEACON_INTERVAL_CODE, -2);
op_intrpt_priority_set (OPC_INTRPT_SELF, BACKOFF_EXPIRATION_CODE, 15); // the highest priority
```

`wban_mac_init()`为 MAC 层模块初始化函数，主要用于读取在节点和 MAC 模块处设定的参数，区分当前节点是 Hub 还是 Node. 如果是 Hub 则读取超帧相关参数，若为 Node 则读取 `Connection Request` 参数。最后设置统计量参数及协议状态参数，较为重要的参数有：

1. `waitForACK` - 是否在等待对方的 ACK?
2. `TX_ING` - 是否正处于包传输过程中？如果是的话就不能再向物理广播模块发送包。
3. `attemptingToTX` - 一切都已就绪，等待将包传输至物理层。可以理解为同步锁，某个包已获得发送至物理层的权利，其他包再要发就得等资源释放出来。

`op_intrpt_priority_set`用于设置中断的优先级，因为如果多个自中断在同一时刻同时产生时，中断的处理是有一定顺序区分的，故需要妥善设置不同中断的优先级。

## `wait_beacon`等待beacon

`wait_beacon`状态机入口代码如下，主要用于区分当前节点是否为Hub, 若为 Hub 则周期性广播 beacon 帧。若为 普通节点 Node 则接收其他节点发送的包，接收包的入口函数为 `wban_parse_incoming_frame`, 接收包之后即转向下一个状态机`idle`。

```
/************************************************************************
In this state a node check if it is a Hub:
- Node is a Hub : it sends beacon frame interval
- Node is a Node : it waits until a beacon frame from the network. 
*/
    
/* get the packet from the stream */
switch (op_intrpt_type()) {
	case OPC_INTRPT_STRM : 
	{
		wban_parse_incoming_frame ();
		op_intrpt_schedule_self (op_sim_time (), DEFAULT_CODE); // -> to IDLE state
		break;
	};
	default :
	{
	};
}
```

## `idle`模块

`idle`模块仅有出口执行代码，代码也就一行。

```
/* Call the interrupt processing routing for each interrupt */
wban_mac_interrupt_process();
```

用于捕捉所有中断，整个状态机跳转自此全部由代码接管，不显式出现于状态机中。显示使用状态机相对好懂一些，但暂时精力有限，也就不打算继续做大的改动了。

## 具体函数解析

### `wban_parse_incoming_frame` - 解析接收到的包

以下代码有删减，只是把一些核心代码和逻辑贴出来了。

```c
/*--------------------------------------------------------------------------------
 * Function:	wban_parse_incoming_frame
 *
 * Description:	parse the incoming packet and make the adequate processing
 *
 * No parameters
 *--------------------------------------------------------------------------------*/
static void wban_parse_incoming_frame() {
	/* get the packet from the input stream	*/
	Stream_ID = op_intrpt_strm();
	rcv_frame = op_pk_get (Stream_ID);
	
	frame_type_fd = 0;
	/* check from what input stream the packet is received and do the right processing*/
	switch (Stream_ID) {
		case STRM_FROM_RADIO_TO_MAC: /*A PHY FRAME (PPDU) FROM THE RADIO RECIEVER*/
			ppdu_bits = op_pk_total_size_get(rcv_frame);
			/* get MAC frame (MPDU=PSDU) from received PHY frame (PPDU)*/
			op_pk_nfd_get_pkt (rcv_frame, "PSDU", &frame_MPDU);
			ete_delay = op_sim_time() - op_pk_creation_time_get(frame_MPDU);
			if (MAC_SLEEP == mac_state){
				FOUT;
			}
			op_pk_nfd_get (frame_MPDU, "BAN ID", &ban_id);
			op_pk_nfd_get (frame_MPDU, "Recipient ID", &recipient_id);
			op_pk_nfd_get (frame_MPDU, "Sender ID", &sender_id);

			// filter the incoming BAN packet - not implemented entirely
			/*update the battery module*/
			if (!is_packet_for_me(frame_MPDU, ban_id, recipient_id, sender_id)) {
				FOUT;
			}

			/* repalce the mac_attr.receipient_id with Sender ID */
			mac_attr.recipient_id = sender_id;
			if(I_ACK_POLICY == ack_policy_fd){
				ack_seq_num = sequence_number_fd;
				op_intrpt_schedule_self(op_sim_time()+pSIFS, SEND_I_ACK);
				if(ack_seq_nid[sender_id%NODE_MAX] != ack_seq_num){
					ack_seq_nid[sender_id%NODE_MAX] = ack_seq_num;
				}else{
					// printf("\t  Duplicate packet received\n");
					op_pk_destroy (rcv_frame);
					op_pk_destroy (frame_MPDU);
					FOUT;
				}
			}

			switch (frame_type_fd) {
				case DATA: /* Handle data packets */
					// printf ("\t  Data Packet reception from sender_id=%d\n", sender_id);
					/* collect statistics */
					op_pk_nfd_get_pkt (frame_MPDU, "MAC Frame Payload", &frame_MSDU);
					app_latency = op_sim_time() - op_pk_creation_time_get(frame_MSDU);
					latency_avg[frame_subtype_fd] = (latency_avg[frame_subtype_fd] * data_stat_local[frame_subtype_fd][RCV].number + ete_delay)/(data_stat_local[frame_subtype_fd][RCV].number + 1);
					data_stat_local[frame_subtype_fd][RCV].number += 1;
					data_stat_local[frame_subtype_fd][RCV].ppdu_kbits += 0.001*ppdu_bits;
					wban_extract_data_frame (frame_MPDU);
					/* send to higher layer for statistics */
					op_pk_send (frame_MPDU, STRM_FROM_MAC_TO_SINK);
					break;
				case MANAGEMENT: /* Handle management packets */
					// op_stat_write(stat_vec.data_pkt_rec, 0.0);
					// printf ("\t  Management Packet reception\n");
					switch (frame_subtype_fd) {
						case BEACON: 
							// printf ("\t    Beacon Packet reception\n");
							wban_extract_beacon_frame (frame_MPDU);
							break;
						case CONNECTION_REQUEST:
							// printf ("\t    Connection Request Packet reception\n");
							wban_extract_conn_req_frame (frame_MPDU);
							break;
						case CONNECTION_ASSIGNMENT:
							// printf ("\t    Connection Assignment Packet reception\n");
							wban_extract_conn_assign_frame (frame_MPDU);
							break;
					}
					break;
				case CONTROL: /* Handle control packets */
					// printf ("\t  Control Packet reception\n");
					switch (frame_subtype_fd) {
						case I_ACK:
							// printf ("\t    I-ACK Packet reception\n");
							wban_extract_i_ack_frame (frame_MPDU);
							break;
						case BEACON2: 
							// printf ("\t    BEACON2 Packet reception\n");
							wban_battery_sleep_start(mac_state);
							mac_state = MAC_SLEEP;
							wban_extract_beacon2_frame(frame_MPDU);
							break;
					}
					break;
				default:	/*OTHER FRAME TYPES*/
					// printf("\t  Other Packet reception\n");
					break;
			}
			break;
		case STRM_FROM_TRAFFIC_UP_TO_MAC: /* INCOMMING PACKETS(MSDU) FROM THE TRAFFIC SOURCE */
			wban_encapsulate_and_enqueue_data_frame (rcv_frame, I_ACK_POLICY, node_attr.traffic_dest_id);			
			break;
		default : break;
	}
	op_pk_destroy (rcv_frame);
	/* Stack tracing exit point */
	FOUT;
}
```

首先从流中断中获取流ID，并从流ID中获取接收到的包。
```c
Stream_ID = op_intrpt_strm();
rcv_frame = op_pk_get (Stream_ID);
```

从流ID中区分其是哪一种中断类型，对于 MAC 层模块来说，其接收到的包要么来自上层应用层，要么来自底层物理层，如果都不是，则删除此流中断中携带的包。
```
switch (Stream_ID)
	case STRM_FROM_RADIO_TO_MAC:
	case STRM_FROM_TRAFFIC_UP_TO_MAC:
	op_pk_destroy (rcv_frame);
```

解包来自物理层发送过来的包，PPDU ==> PSDU(对应于 MAC 层中的MPDU)，计算端到端时延，并提取出 BAN ID, Recipient ID, Sender ID用于函数`is_packet_for_me`入口参数。若不满足节点当前接收状态或者接收到的包不是发给自己的就丢弃之。若要自定义包过滤规则可以更改函数`is_packet_for_me`.
```c
ete_delay = op_sim_time() - op_pk_creation_time_get(frame_MPDU);
if (MAC_SLEEP == mac_state){
	FOUT;
}
op_pk_nfd_get (frame_MPDU, "BAN ID", &ban_id);
op_pk_nfd_get (frame_MPDU, "Recipient ID", &recipient_id);
op_pk_nfd_get (frame_MPDU, "Sender ID", &sender_id);

// filter the incoming BAN packet - not implemented entirely
/*update the battery module*/
if (!is_packet_for_me(frame_MPDU, ban_id, recipient_id, sender_id)) {
	FOUT;
}
```

检查包的 I_ACK_POLICY, 如果是要求返回 I-ACK，则应在 pSIFS 之后发送 I-ACK, 同时检查当前包是否为重发的包，若为已经接收过多包则丢弃并返回。
```c
/* repalce the mac_attr.receipient_id with Sender ID */
mac_attr.recipient_id = sender_id;
if(I_ACK_POLICY == ack_policy_fd){
	ack_seq_num = sequence_number_fd;
	op_intrpt_schedule_self(op_sim_time()+pSIFS, SEND_I_ACK);
	if(ack_seq_nid[sender_id%NODE_MAX] != ack_seq_num){
		ack_seq_nid[sender_id%NODE_MAX] = ack_seq_num;
	}else{
		// printf("\t  Duplicate packet received\n");
		op_pk_destroy (rcv_frame);
		op_pk_destroy (frame_MPDU);
		FOUT;
	}
}
```

处理完 I-ACK 的包之后即对 MAC 层包进行过滤分类处理，主要分为三大类—— DATA, MANAGEMENT, CONTROL. 

1. 对于数据包主要做一些统计工作，如时延等。
2. 对于管理帧则需要进一步区分到底是哪一类，主要分为 BEACON, CONNECTION_REQUEST 和 CONNECTION_ASSIGNMENT 三类。
    - 对于 BEACON 帧，则调用`wban_extract_beacon_frame`对其进行进一步解析。
    - 对于 CONNECTION_REQUEST 帧，则调用`wban_extract_conn_req_frame`对其进行进一步解析。
    - 对于 CONNECTION_ASSIGNMENT 帧，则调用`wban_extract_conn_assign_frame`对其进行进一步解析。
3. 对于控制帧主要分为两类，I-ACK 和 BEACON2.
    - 对于 I-ACK 则调用`wban_extract_i_ack_frame`.
    - 对于 BEACON2 则调用`wban_extract_beacon2_frame`.
```c
switch (frame_type_fd) {
	case DATA: /* Handle data packets */
		/* collect statistics */
		wban_extract_data_frame (frame_MPDU);
		/* send to higher layer for statistics */
		op_pk_send (frame_MPDU, STRM_FROM_MAC_TO_SINK);
	case MANAGEMENT: /* Handle management packets */
		switch (frame_subtype_fd) {
			case BEACON: 
				wban_extract_beacon_frame (frame_MPDU);
			case CONNECTION_REQUEST:
				wban_extract_conn_req_frame (frame_MPDU);
			case CONNECTION_ASSIGNMENT:
				wban_extract_conn_assign_frame (frame_MPDU);
		}
	case CONTROL: /* Handle control packets */
		switch (frame_subtype_fd) {
			case I_ACK:
				wban_extract_i_ack_frame (frame_MPDU);
			case BEACON2:
				wban_battery_sleep_start(mac_state);
				mac_state = MAC_SLEEP;
				wban_extract_beacon2_frame(frame_MPDU);
		}
```

接收来自应用层模块发送的包，主要由`case STRM_FROM_TRAFFIC_UP_TO_MAC`负责，具体封装函数为`wban_encapsulate_and_enqueue_data_frame`, 目前的策略为均使用 I-ACK 策略，如果要调整 ACK 策略可从这里更改，也可在发送到物理层之前更改，不过在发送至物理层之前更改需要注意修改包后的端到端时延计算。

```c
case STRM_FROM_TRAFFIC_UP_TO_MAC: /* INCOMMING PACKETS(MSDU) FROM THE TRAFFIC SOURCE */
	wban_encapsulate_and_enqueue_data_frame (rcv_frame, I_ACK_POLICY, node_attr.traffic_dest_id);
```

### `wban_log_file_init`

用于初始化记录各项统计数据的文件，目前该仿真平台并未采用 OPNET 官方的统计量记录方式，个人认为设置探针等方式对于开发阶段的调试过于麻烦(需要先手动选定仿真场景和数据，再将数据导出至 EXCEL, 最后再统计)，嗯，我不喜欢拿鼠标戳来戳去的低效做法，借鉴了 Castalia 中的方法，直接采用一个文本记录所有数据，后期使用 Python 进行数据提取和分析。但这种方式的缺点是用到了文件 IO，且 OPNET 中不同的节点在仿真过程中即为不同的进程，对同一文件读写需要加锁，故频繁记录大量数据时会导致仿真速度急剧下降。我的做法是映射一部分内存作为文件读写，一定程度上可以改善，但是由于频繁加锁解锁的存在，仿真速度还是不够理想。另一种思路就是不同进程读写不同文件，这样就不必频繁加锁解锁了，这样后期处理下就好了，Python 大法好！

函数`wban_log_file_init`主要利用了当前时间和在节点处设置的目录进行自动命名，便于后期使用 Python 自动化分析，同时防止多次仿真时文件被覆盖。
```c
static void wban_log_file_init() {
	op_ima_obj_attr_get (node_attr.objid, "Log File Directory", dir_path);
	op_ima_obj_attr_get (node_attr.objid, "Log Level", &log_level);

	time(&rawtime);
	p=localtime(&rawtime);
	// strftime(buffer, 30, "%Y-%m-%d_%H-%M-%S", p);
	// strftime(buffer, 30, "%Y-%m-%d_%H-%M", p);
	strftime(buffer, 30, "%Y-%m-%d", p);
	for(i=0; i<(sizeof(dir_path)/sizeof(dir_path[0])); i++){
		if (dir_path[i] == '\0'){
			break;
		}
	}
	if(prg_file_path_create(dir_path, PRGC_FILE_PATH_CREATE_OPT_DIRECTORY) == PrgC_Compcode_Failure){
		op_sim_end("ERROR : Log File is not valid.","INVALID_FILE", "","");
	}
	if(dir_path[i-1] == '\\'){
		sprintf(log_name, "%s%s-ver%d.trace", dir_path, buffer, node_attr.protocol_ver);
	}else{
		sprintf(log_name, "%s\\%s-ver%d.trace", dir_path, buffer, node_attr.protocol_ver);
	}
	/* verification if the dir_path is a valid directory */
	if (prg_path_name_is_dir (dir_path) == PrgC_Path_Name_Is_Not_Dir) {
		op_sim_end("ERROR : Log File Directory is not valid directory name.","INVALID_DIR", "","");
	}
}
```

### `is_packet_for_me`

包检测的入口函数，判断从物理层接收到的包是否是自己需要接收的。主要从目前所处的状态(比如是否睡眠)和是否为本网内节点发送的包进行过滤。这个函数对于多网环境和组簇的场景比较重要。主要根据入口参数如 ban_id, recipient_id, sender_id判断是否需要接收包。

1. 首先判断 sender_id 是否就是自身，如果为自身则直接丢弃，OPNET 中节点不仅可以接收来自其他节点的包，还可以接收来自自身的包，一般来说自身的包需要直接丢弃。
2. 接着判断 ban_id 是否和自身 ban_id 相同，不同则代表收到的包来自其他网络，一般来说直接丢弃，但若有特殊用途则做进一步处理。
3. 判断 recipient_id 为单播还是广播。

```c
static Boolean is_packet_for_me(Packet* frame_MPDU, int ban_id, int recipient_id, int sender_id) {
	/*Check if the frame is loop*/
	if (mac_attr.sender_id == sender_id) {
		op_pk_destroy (frame_MPDU);
	}
	if (node_attr.ban_id != ban_id) {
	}
	if ((mac_attr.sender_id == recipient_id) || (BROADCAST_NID == recipient_id)) {
	} else {
		op_pk_destroy (frame_MPDU);
	}
}
```

### `wban_send_beacon_frame`

发送 Beacon 帧函数，由 Hub 节点进行发送，读取在 Hub 节点处设置的 Beacon 帧属性，将这些帧属性信息封装至包中。封装顺序为 MSDU ==> MPDU ==> PPDU. 在发送 Beacon 帧之前获得当前超帧的起始时间`SF.BI_Boundary`，便于后面设置定时器和超帧起止时间信息。如果是第一次发送 beacon 帧，还需要根据`init_flag`设置其他变量信息。

```c
static void wban_send_beacon_frame () {
	beacon_MSDU = op_pk_create_fmt ("wban_beacon_MSDU_format");
	/* set the fields of the beacon frame */
	/* create a MAC frame (MPDU) that encapsulates the beacon payload (MSDU) */
	beacon_MPDU = op_pk_create_fmt ("wban_frame_MPDU_format");
	op_pk_nfd_set_pkt (beacon_MPDU, "MAC Frame Payload", beacon_MSDU); // wrap beacon payload (MSDU) in MAC Frame (MPDU)

	SF.BI_Boundary = op_pk_creation_time_get (beacon_MPDU);
	if(init_flag){
	}
	SF.current_slot = (int)(TX_TIME(wban_norm_phy_bits(beacon_MPDU), node_attr.data_rate)/SF.slot_sec);

	/* send the MPDU to PHY and calculate the energy consuming */
	wban_send_mac_pk_to_phy(beacon_MPDU);
}
```

### `wban_send_beacon2_frame`

发送 beacon2 控制帧用，如果要在超帧中设置 CAP 时需要发送此帧，同时也可以按需更改 beacon2 帧内容。如之前实现了两种 MAC 协议，其中就充分利用了 beacon2 的信息，所以就可以在此函数中根据不同协议做进一步区分。如果要在 beacon2 部分中做复杂的时隙再分配可以仔细看看这个函数的实现。正常情况下不需要利用到这个函数。

### `wban_extract_conn_req_frame`

主要用于 Hub 解析接收到的 Connection Request 帧，Hub 判断是否给相应 Node 分配时隙和告知其他资源。Node 在 Scheduling 阶段需要请求的时隙数为 `allocation_length`, Hub 正是通过此帧获知这一信息。不过感觉这有点不科学，如果 Node 需要的时隙数是变化的如何破？重新发送 Connection Request 帧的开销太大了。具体实现时可以根据自己的需求灵活更改。


### `wban_extract_conn_assign_frame`

用于 Node 接收 Hub 发送过来的 Conenction Assignment 帧，Hub 在此帧中告知 Node 在超帧 Superframe 中的起止时隙处。如 Interval Start 和 Interval End。目前该仿真平台对于 Connection Request 和 Connection Assignment 均只做一次，故改变了 init_flag 变量。

### `wban_extract_beacon_frame`

用于 Node 接收 Hub 发送过来的 beacon 帧，获知超帧中各阶段的起止信息和同步信息。如果当前 Node 并未加入到当前 WBAN 网络，则将`UNCONNECTED_NID`更改为合适的 NID。如果需要组簇等实现其他算法，这里可能需要做一些改动。最后调用`wban_schedule_next_beacon`等待接收下一个超帧。

### `wban_extract_beacon2_frame`

用于 Node 接收 Hub 发送的 beacon2 控制帧，如果需要使用 CAP 和 MAP2 阶段才需要关注此函数。

### `wban_schedule_next_beacon`

用于 Node 和 Hub 根据 beacon 帧中相应信息设置当前超帧各阶段的起止时间和安排下一超帧的起始时间。

### `wban_send_conn_req_frame`

用于 Node 根据自身节点属性(如在 Scheduling 阶段需要多长时隙)向 Hub 发送 Connection Request 帧。目前的做法是仅与 Hub 成功交互一次。

### `wban_send_conn_assign_frame`

Hub 接收到 Connection Request 帧后即准备安排发送此帧，用于确认 Node 在 Scheduling 阶段的起止时隙和其他信息。

### `wban_send_i_ack_frame`

用于 Hub 和 Node 向需要返回 I-ACK 的包回复 I-ACK 包，注意需要判断是否越界——`ack_sent < phase_end_timeG`.

### `wban_encapsulate_and_enqueue_data_frame`

用于封装来自应用层发送过来的包，封装之前先确认当前 MAC 模块的状态：

```
if ((MAC_SLEEP != mac_state) && (MAC_SETUP != mac_state))
```

### `wban_extract_data_frame`

用于解析收到的数据帧，如果需要在数据帧中捎带信息则需要更改此函数。

### `wban_extract_i_ack_frame`

用于解析接收到的 I-ACK 帧，这个函数有点微妙，首先需要判断当前是不是在等待其他节点回复 I-ACK 。

1. `pkt_to_be_sent.enable`: 当前是否即将发送包？是的话就不需要当前的 I-ACK 包。
2. `pkt_tx_total`: 忘了...
3. `waitForACK`: 是否在等待接收 I-ACK。

三者有一为真即可接收此 I-ACK 包，这种处理方式的正确性还有待考证。

```c
if((!pkt_to_be_sent.enable) || (0 == pkt_tx_total) || (!waitForACK))
```

如果当前状态为正在等待接收 I-ACK, 则需要关闭最近的一次等待 I-ACK 结束的定时器，具体则是`WAITING_ACK_END_CODE`中断。关闭定时器的大致形式如下：

```
op_intrpt_disable (OPC_INTRPT_SELF, WAITING_ACK_END_CODE, OPC_TRUE);
```

如果当前接收到的为数据帧，则统计相应变量信息。最后设置相应全局变量信息用于加锁释放锁等操作。

```
waitForACK = OPC_FALSE;
TX_ING = OPC_FALSE;
// attemptingToTX = OPC_FALSE;
pkt_to_be_sent.enable = OPC_FALSE;
pkt_tx_total = 0;
pkt_tx_out_phase = 0;
pkt_tx_fail = 0;
```

以下就一些状态信息做一些说明：

- `waitForACK`: 是否在等待接收 I-ACK
- `TX_ING`: 是否正处于发送包的过程当中
- `pkt_to_be_sent.enable`: 此时是否可以发送包？
- `pkt_tx_total`: 忘了，待补充
- `pkt_tx_out_phase`: 同上
- `pkt_tx_fail`: 同上

最后在 pSIFS 时间间隔后设置可发送下一个包。

```
op_intrpt_schedule_self (op_sim_time() + pSIFS, TRY_PACKET_TRANSMISSION_CODE);
```

### `wban_attempt_TX`

在所有可能的阶段尝试传输，粗略来讲分为两种传输模式——重传原来的包和从 MAC 层缓存队列中取出新包进行传输。
能进行传输的前提是以下三个条件均为假。

1. waitForACK: 代表当前节点正在等待接收 I-ACK，不可进行传输包的操作。
2. attemptingToTX: 已经有包即将传输，故不安排新的包。
3. TX_ING: 有包正在传输过程中，不可安排其他包传输。

```
if(waitForACK || attemptingToTX || TX_ING)
```

如果达到传输要求，则进一步判断当前 MAC 层所处状态，如是处于 MAP(只能使用 Scheduling) 还是 CAP(使用CSMA/CA)阶段。

```
SF.IN_CAP_PHASE = OPC_TRUE;
SF.IN_MAP_PHASE = OPC_FALSE;
```

保险起见，检查当前节点是否还处于传输阶段 - `op_stat_local_read(TX_BUSY_STAT)`.

pkt_to_be_sent.enable: 判断当前节点是否被允许发送包。
pkt_tx_total < max_packet_tries: 当前包传输的次数是否超过最大重传次数。

```
if((pkt_to_be_sent.enable) && (pkt_tx_total < max_packet_tries))
```

if(!can_fit_TX(&pkt_to_be_sent)): 包被取出来但是如果传输此包则会超出当前允许的时隙边界

如果当前节点处于 CAP 阶段，则调用 `wban_attempt_TX_CSMA` 使用 CSMA/CA 进行传输。
如果当前节点处于 MAP 阶段(注意在具体的代码实现中 MAP 仅是指当前节点在整个超帧中所分配的部分起止时隙段，并不是整个 MAP)，使用`wban_send_mac_pk_to_phy(frame_MPDU_to_be_sent);`将 MAC 层的包送至 PHY 进行封装。

随后检查 MAC 层子队列(管理帧和数据帧)是否非空，优先发送管理帧。

若`pkt_to_be_sent.enable`为真，代表当前包可发送，若处于 CAP 阶段，则设置 CSMA 相应的参数并调用 `wban_attempt_TX_CSMA();`. 若处于 MAP 则直接传至 PHY。

### `wban_attempt_TX_CSMA`

使用 CSMA 接入信道前的参数设置，如根据不同用户优先级初始化不同的退避窗口长度。

设置全局状态变量`attemptingToTX = OPC_TRUE;`，表明已经有包处于待传输状态，不再从 MAC 层子队列中获取新的包进行传输。最终产生`CCA_START_CODE`中断。

### `wban_backoff_period_boundary_get`

求出退避时相应的时隙边界，使用 slot based CSMA 时需要此函数，默认应该是采用这种方式。

### `wban_backoff_delay_set`

设置对不同用户优先级包退避时的时延，退避到期的时间，当前阶段剩余的时间。

### `can_fit_TX`

接收包属性指针参数，获取 ACK 的策略，判断当前剩余时间是否满足传输此包和 I-ACK 的需求。若为 N-ACK 策略，则不需要等待接收 ACK，若设置为 I-ACK 策略，则还需要计算接收 I-ACK 的时间。

### `wban_send_mac_pk_to_phy`

发送 MAC 层的包到物理层，此时已确定可以发送此包，需要设置如下两个变量：

```
attemptingToTX = OPC_FALSE;
TX_ING = OPC_TRUE;
```

对于 I-ACK 策略的包，需要设置等待接收 ACK 结束时的中断。

```
op_intrpt_schedule_self(ack_expire_time, WAITING_ACK_END_CODE);
```

### `wban_norm_phy_bits`

归一化 MAC 帧到物理层包，考虑了物理层头信息。

### `phy_to_radio`

发包的最后一个环节，将封装好的物理层帧送至无线广播模块。

## MAC 层核心逻辑 - `wban_mac_interrupt_process`

MAC 层核心模块！用于处理各种中断及跳转。

对于流中断`OPC_INTRPT_STRM`，则调用`wban_parse_incoming_frame`进行解析。对于自中断`OPC_INTRPT_SELF`则分为以下多种类型的中断。

1. `BEACON_INTERVAL_CODE` - beacon 帧间隔中断码，用于实现超帧及发送/接收beacon。
2. `INCREMENT_SLOT` - 根据设定的 slot 长度周期性地自增 slot, 使用最简易的定时器实现。
3. `START_OF_EAP1_PERIOD_CODE` - EAP1 阶段的起始中断码，用于处理紧急业务。
4. `START_OF_RAP1_PERIOD_CODE` - RAP1 阶段的起始中断码，用于 CSMA 接入方式，适用于不同用户优先级业务。
5. `START_OF_MAP1_PERIOD_CODE` - MAP1 阶段的起始中断码，用于 Scheudling 接入方式。
6. `SEND_B2_FRAME` - 发送 beacon2 控制帧中断码，开始发送 beacon2 中断码。
7. `START_OF_MAP2_PERIOD_CODE` - MAP2 阶段的起始中断码，几乎和 MAP1 作用相同，一般不需要使用。
8. `START_OF_CAP_PERIOD_CODE` - CAP 阶段的起始中断码，用于 CSMA 接入方式，一般不需要此阶段。
9. `START_OF_SLEEP_PERIOD` - SLEEP 阶段的起始中断码，睡眠开始的标志。
10. `TRY_PACKET_TRANSMISSION_CODE` - 尝试传输包，调用`wban_attempt_TX`.
11. `CCA_START_CODE` - CCA 开始中断码，用于 CSMA 过程。
12. `CCA_EXPIRATION_CODE` CCA 结束中断码，节点根据 CCA 得到信道忙闲情况。
13. `START_TRANSMISSION_CODE` - 开始传输包，将要发送的 MAC 层包送至 PHY。
14. `WAITING_ACK_END_CODE` - 等待 ACK 结束的中断码，判断是否超过最大重传次数等安排重传。
15. `N_ACK_PACKET_SENT` - 将发送的包 ACK 策略设置为 N-ACK 时，此中断代表可以继续发送新的包。

### 统计中断 - `OPC_INTRPT_STAT`

除了 MAC 层本身的自中断，还有 PHY 反馈给 MAC 的统计中断如发送模块忙，接收包冲突等。

### 仿真结束中断 - `OPC_INTRPT_ENDSIM`

仿真结束时需要统计各变量信息。


## MAC helper

1. `queue_status` - 获取 MAC 队列中子队列状态和队列容量等信息。
2. `subq_info_get` - 获取子队列中的信息。
3. `subq_data_info_get` - 获取数据子队列中的信息。
4. `header4mac2phy` - MAC 层到 PHY 需要增加的头部比特数。

## Battery Model

更新能量计算模块的能量信息，使用 ICI 远程中断信息进行交互。同时向能量模块传递 MAC 状态信息。

1. `wban_battery_update_tx` - 计算接收包时所耗费的能量。
2. `wban_battery_update_rx` - 计算接收包时所耗费的能量。
3. `wban_battery_cca` - 计算 CSMA 中 CCA 阶段耗费的能量。
4. `wban_battery_sleep_start` - 睡眠起始时刻。
5. `wban_battery_sleep_end` - 睡眠结束时刻。