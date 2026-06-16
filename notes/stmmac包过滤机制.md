```c
static void dwxgmac2_set_mchash(void __iomem *ioaddr, u32 *mcfilterbits,
				int mcbitslog2)
{
	int numhashregs, regs;

	switch (mcbitslog2) {
	case 6:
		numhashregs = 2;
		break;
	case 7:
		numhashregs = 4;
		break;
	case 8:
		numhashregs = 8;
		break;
	default:
		return;
	}

	for (regs = 0; regs < numhashregs; regs++)
		writel(mcfilterbits[regs], ioaddr + XGMAC_HASH_TABLE(regs));
}

static void dwxgmac2_set_filter(struct mac_device_info *hw,
				struct net_device *dev)
{
	void __iomem *ioaddr = (void __iomem *)dev->base_addr;
	u32 value = readl(ioaddr + XGMAC_PACKET_FILTER);
	int mcbitslog2 = hw->mcast_bits_log2;
	u32 mc_filter[8];
	int i;

	value &= ~(XGMAC_FILTER_PR | XGMAC_FILTER_HMC | XGMAC_FILTER_PM);
	value |= XGMAC_FILTER_HPF;

	memset(mc_filter, 0, sizeof(mc_filter));

	if (dev->flags & IFF_PROMISC) {
		value |= XGMAC_FILTER_PR;
		value |= XGMAC_FILTER_PCF;
	} else if ((dev->flags & IFF_ALLMULTI) ||
		   (netdev_mc_count(dev) > hw->multicast_filter_bins)) {
		value |= XGMAC_FILTER_PM;

		for (i = 0; i < XGMAC_MAX_HASH_TABLE; i++)
			writel(~0x0, ioaddr + XGMAC_HASH_TABLE(i));
	} else if (!netdev_mc_empty(dev) && (dev->flags & IFF_MULTICAST)) {
		struct netdev_hw_addr *ha;

		value |= XGMAC_FILTER_HMC;

		netdev_for_each_mc_addr(ha, dev) {
			u32 nr = (bitrev32(~crc32_le(~0, ha->addr, 6)) >>
					(32 - mcbitslog2));
			mc_filter[nr >> 5] |= (1 << (nr & 0x1F));
		}
	}

	dwxgmac2_set_mchash(ioaddr, mc_filter, mcbitslog2);

	/* Handle multiple unicast addresses */
	if (netdev_uc_count(dev) > hw->unicast_filter_entries) {
		value |= XGMAC_FILTER_PR;
	} else {
		struct netdev_hw_addr *ha;
		int reg = 1;

		netdev_for_each_uc_addr(ha, dev) {
			dwxgmac2_set_umac_addr(hw, ha->addr, reg);
			reg++;
		}

		for ( ; reg < XGMAC_ADDR_MAX; reg++) {
			writel(0, ioaddr + XGMAC_ADDRx_HIGH(reg));
			writel(0, ioaddr + XGMAC_ADDRx_LOW(reg));
		}
	}

	writel(value, ioaddr + XGMAC_PACKET_FILTER);
}
```

# 1. `IFF_PROMISC`(0x100): receive all packets

在该种模式下直接配置 `XGMAC_FILTER_PR | XGMAC_FILTER_PCF`:

- `XGMAC_FILTER_PR`(BIT_0): When this bit is set, the Address Filtering module passes all incoming packets irrespective of the destination or source address. The MAC clears the SA or DA Filter Fail status bits of the Rx Status Word when PR is set.
- `XGMAC_FILTER_PCF`(BIT_7): Pass Control Packets. These bits control the forwarding of all control packets (including unicast and multicast Pause packets):

    - 00: The MAC filters all control packets from reaching the application.
    - 01: The MAC forwards all control packets except Pause packets to the application even if they fail the Address filter.
    - 10: The MAC forwards all control packets to the application even if they fail the Address filter.
    - 11: The MAC forwards the control packets that pass the Address filter.

    此处配置为 0b10, 通过所有 packets.

# 2. `IFF_ALLMULTI`(0x200): receive all multicast packets

- `XGMAC_FILTER_PM`(BIT_4): Pass All Multicast. When this bit is set, it indicates that all the received packets with a multicast destination address (first bit in the destination address field is 1) are passed. When this bit is reset, filtering of multicast packet is done depending on HMC bit.

该流程同步清空了 `XGMAC_HASH_TABLE`.

# 3. `IFF_MULTICAST`(0x8000): supports multicast

- `XGMAC_FILTER_HMC`(BIT_2): Hash Multicast. When this bit is set, the MAC performs the destination address filtering of received multicast packets according to the hash table. When this bit is reset, the MAC performs the perfect destination address filtering for multicast packets, that is, it compares the DA field with the values programmed in MAC_Address registers.

在 HMC 的情况下, 包过滤机制由 DA MAC_Address 配置变更为 hash table 做计算。随后 mac 会根据 hash 做包过滤:

The 64-bit, 128-bit, or 256-bit hash table is used for group address filtering. For hash filtering, the content of the destination address in the incoming packet is passed through the CRC logic and the upper six (seven or eight in 128- or 256-bit Hash) bits of the CRC are used to index the content of the Hash table. The most significant bits determines the register to be used (Hash Table Register X), and the least significant five bits determine the bit within the register For example, a hash value of 7b'1100000 (in 128-bit Hash) selects Bit 0 of the Hash Table Register 3 and a value of 8b'10111111 (in 256-bit Hash) selects Bit 31 of the Hash Table Register 5.
The hash value of the destination address is calculated in the following way:

- Calculate the 32-bit CRC for the DA (See IEEE 802.3-2018, Section 3.2.8 for the steps to calculate
CRC32).
- Perform bit-wise reversal for the value obtained in Step 1.
- Take the upper 6 (or 7 or 8) bits from the value obtained in Step 2.

If the corresponding bit value of the MAC_Hash_Table_Reg0 register is 1'b1, the packet is accepted. Otherwise, it is rejected. If the PM bit is set in MAC_Packet_Filter, all multicast packets are accepted regardless of the multicast hash values.

If the Hash Table register is configured to be double-synchronized to the (X)GMII clock domain, the synchronization is triggered only when Bits[31:24] (in little-endian mode) or Bits[7:0] (in big-endian mode) of the Hash Table Register X registers are written.

If double-synchronization is enabled, consecutive writes to this register must be performed after at least four clock cycles in the destination clock domain.
