+++
date = '2025-10-30T17:53:14+08:00'
draft = false
title = 'My First Post'

summary = '这是一个测试摘要信息'

+++

## 测试页面

### 111

#### 333

1111111111





``` java
private void computXolRiskData(GcRiClaimVo gcRiClaimVo, GrXTtySectResVo grXTtySectResVo, List<GcRiClaimXTty> oldGcRiClaimXTtyList, List<GcRiClaimXTtyVo> gcRiClaimXTtyList, Map<String, BigDecimal> map) throws Exception {
		// 币别进行兑换
		BigDecimal exrate = this.getExchangeRate(grXTtySectResVo.getCurrency(), gcRiClaimVo.getRicurrency(), null, null);
		BigDecimal excessLoss = grXTtySectResVo.getExcessLoss();// 起赔额
		BigDecimal contQuota = grXTtySectResVo.getContQuota();// 层高
		BigDecimal curtotLimit = grXTtySectResVo.getCurtotLimit();// 当前层总限额
		BigDecimal alreadyPaid = BigDecimal.ZERO;//已经出帐的金额，前次再保已决赔款汇总 更新时点：出账时 取Sum（出过账的RIPaid）
		BigDecimal paid = BigDecimal.ZERO;// 应摊回赔款
		BigDecimal paidChg = null;
		BigDecimal retenPaid = null;
		BigDecimal osReserve = null;
		BigDecimal osReserveChg = null;
		BigDecimal grossIncurred = null;
		BigDecimal retenGrossIncurred = null;
		BigDecimal grossIncurredChg = BigDecimal.ZERO;
		String preXolNo = null;
		Date preTime = null;
		BigDecimal oldPaidEffective = null;

		Map<String, BigDecimal> mapAmount = this.getClaimAmount(gcRiClaimVo, grXTtySectResVo.getCurrency());
		// 自留已决
		retenPaid = mapAmount.get("retenPaid");
		// 自留毛损失
		retenGrossIncurred = mapAmount.get("retenGrossIncurred");
		BigDecimal osExrate = mapAmount.get("osExrate");

		Boolean b = false;// 是否用层累积限额
		Boolean isNotice = true;// 是否用于通知书

		//1.计算限额
		if (null != curtotLimit) {
			b = true;
			// 合约限额-已经出帐生成分摊信息的金额
			curtotLimit = curtotLimit.subtract(map.get(grXTtySectResVo.getTtyId()));
		} else {
			curtotLimit = contQuota;
		}

		GcRiClaimXTty oldGcRiClaimXTty = null;
		if (null != oldGcRiClaimXTtyList && oldGcRiClaimXTtyList.size() > 0) {
			for (GcRiClaimXTty g : oldGcRiClaimXTtyList) {
				if(g.getCurrentVerInd()){
					preXolNo = g.getXolNo();
					preTime = g.getTransDate();
					alreadyPaid = g.getAlreadyPaid();
					// 计算有效上期已决：若已出账但未新建版本（billInd='1'），用 paid - modiPaid；否则用 paid
					oldPaidEffective = g.getPaid() == null ? BigDecimal.ZERO : g.getPaid();
					if ("1".equals(g.getBillInd())) {
						BigDecimal oldModi = g.getModiPaid() == null ? BigDecimal.ZERO : g.getModiPaid();
						oldPaidEffective = oldPaidEffective.subtract(oldModi);
						if (oldPaidEffective.compareTo(BigDecimal.ZERO) < 0) {
							oldPaidEffective = BigDecimal.ZERO;
						}
					}
					if(b) {
						curtotLimit = curtotLimit.add(oldPaidEffective).add(g.getAlreadyPaid() == null ? BigDecimal.ZERO : g.getAlreadyPaid());
					}
					if(null != g.getModExcessloss()) {
						excessLoss = g.getModExcessloss();
					}
					if(null != g.getModcontQuota()) {
						contQuota = g.getModcontQuota();
						if(!b) {
							curtotLimit = contQuota;
						}
					}
					oldGcRiClaimXTty = g;
				}
			}
		}



		if (curtotLimit.compareTo(BigDecimal.ZERO) > 0) {

			// 层高大于合约限额，则取合约限额计算
			if (curtotLimit.compareTo(contQuota) < 0) {
				contQuota = curtotLimit;
			}

			// 判断自留是否超过起赔额
			if (retenPaid.compareTo(excessLoss) > 0) {
				LOG.debug("------自留：" + retenPaid + "大于起赔点：" + excessLoss + "------");
				paid = retenPaid.subtract(excessLoss);
				if (paid.compareTo(contQuota) > 0) {
					paid = contQuota;
				}
			}
		}

		grossIncurred = retenGrossIncurred.subtract(excessLoss);
		if (grossIncurred.compareTo(contQuota) > 0) {
			grossIncurred = contQuota;
		}

		//2.计算是否生成数据
		GrXTtyLayerResVo grXTtyLayer = this.requestGrXTtyLayer(grXTtySectResVo.getTtyId(), grXTtySectResVo.getSectId());

		//1. riversion作为轨迹版本号
		//2.进入超赔合约后，gcriclaimxtty.osReserve，版本号加1
		//4.gcriclaimxtty.grossIncurredchg取gcriclaimxtty.grossIncurred的变化量
		//5.生成帐单后，新插一条数据，paid=上一版本paid-上一版本modipaid，modipaid=paid
		//自留估损-层起赔点和层高取小

		if(null == oldGcRiClaimXTty && grossIncurred.compareTo(BigDecimal.ZERO) <= 0) {
			BigDecimal base = "1".equals(grXTtyLayer.getLayerNo()) ? new BigDecimal(0.75) : new BigDecimal(1);
			if (retenGrossIncurred.compareTo(excessLoss.multiply(base)) >= 0){
				grossIncurred = BigDecimal.ZERO;
				isNotice = false;
			}
		}

		if(null != oldGcRiClaimXTty && grossIncurred.compareTo(BigDecimal.ZERO) < 0) {
			grossIncurred = BigDecimal.ZERO;
		}

		GcRiClaimXTtyVo gcRiClaimXTty = null;
		String updateType = "1";
		Boolean isC = false;//是否生成新数据
		Boolean isUp = false;//是否更新旧数据
		osReserve = grossIncurred.subtract(alreadyPaid);
		if(paid.compareTo(BigDecimal.ZERO) < 0) {
			paid = BigDecimal.ZERO;
		}
		paid = paid.subtract(alreadyPaid);
		if(grossIncurred.compareTo(BigDecimal.ZERO) > 0 || (null != oldGcRiClaimXTty && grossIncurred.compareTo(BigDecimal.ZERO) == 0)
				|| (!isNotice && grossIncurred.compareTo(BigDecimal.ZERO) == 0)) {
			if(null != oldGcRiClaimXTty) {
				gcRiClaimXTty = Pages.convert(oldGcRiClaimXTty, GcRiClaimXTtyVo.class);
				BigDecimal oldGrossIncurred = oldGcRiClaimXTty.getGrossIncurred() == null ? BigDecimal.ZERO : oldGcRiClaimXTty.getGrossIncurred();
				BigDecimal oldAlready = oldGcRiClaimXTty.getAlreadyPaid() == null ? BigDecimal.ZERO : oldGcRiClaimXTty.getAlreadyPaid();
				BigDecimal oldOsReserveEffective = oldGrossIncurred.subtract(oldAlready);
				if (oldOsReserveEffective.compareTo(BigDecimal.ZERO) < 0) {
					oldOsReserveEffective = BigDecimal.ZERO;
				}
				osReserveChg = osReserve.subtract(oldOsReserveEffective);
				grossIncurredChg = grossIncurred.subtract(oldGcRiClaimXTty.getGrossIncurred());
				paidChg = paid.subtract(oldPaidEffective == null ? BigDecimal.ZERO : oldPaidEffective);
				//估损有变化，生成一条新数据
				if(osReserveChg.compareTo(BigDecimal.ZERO) != 0) {
					isC = true;
					isUp = true;
				} else if(paid.compareTo(oldPaidEffective == null ? BigDecimal.ZERO : oldPaidEffective) != 0) {
					isC = true;
					//估损没有变化赔付有变化更新本数据，未出帐，则先删后插，已出帐，则插数
					if(!"1".equals(oldGcRiClaimXTty.getBillInd())) {
						grossIncurredChg = grossIncurredChg.add(oldGcRiClaimXTty.getGrossIncurredChg());
						paidChg = paidChg.add(oldGcRiClaimXTty.getPaidChg());
						updateType = "2";
						gcRiClaimXTtyDao.deleteById(oldGcRiClaimXTty.getRiClaimXTtyId());
					} else {
						LOG.debug("估损没有变化赔付有变化更新本数据" + paid.subtract(oldGcRiClaimXTty.getPaid()));
						isUp = true;
					}
				}

				if(isUp) {
					gcRiClaimXTty.setRiVersion(oldGcRiClaimXTty.getRiVersion() + 1);
					gcRiClaimXTty.setXolNo(null);
					FieldUtils.setAllVoValue(oldGcRiClaimXTty, "2", "system");
					oldGcRiClaimXTty.setCurrentVerInd(false);
					gcRiClaimXTtyDao.updateById(oldGcRiClaimXTty);
				}
			} else {
				isC = true;
				gcRiClaimXTty = new GcRiClaimXTtyVo();
                gcRiClaimXTty.setRetentionRate(grXTtySectResVo.getRetentionRate());
				gcRiClaimXTty.setTtyCode(grXTtyLayer.getTtyCode());
				gcRiClaimXTty.setClaimNo(gcRiClaimVo.getClaimNo());
				gcRiClaimXTty.setRiVersion(1);
				gcRiClaimXTty.setTtyId(grXTtySectResVo.getTtyId());
				gcRiClaimXTty.setTtySectId(grXTtySectResVo.getSectId());
				grossIncurredChg = grossIncurred;
				paidChg = paid;
			}
		}

		LOG.debug("------是否生成新数据：" + isC + " grossIncurred：" + grossIncurred + "；paid：" + paid+" excessLoss：" +excessLoss+"os:"+grossIncurred.subtract(alreadyPaid));
		if (isC) {
			gcRiClaimXTty.setOsExchange(osExrate);
			gcRiClaimXTty.setRiClaimXTtyId(Tools.getId());
			gcRiClaimXTty.setRenPrem(null);
			gcRiClaimXTty.setModiRenPrem(null);
			gcRiClaimXTty.setAlreadyPaid(alreadyPaid);
			gcRiClaimXTty.setOsReserve(osReserve);
			gcRiClaimXTty.setPaid(paid);
			gcRiClaimXTty.setModiPaid(paid);
			gcRiClaimXTty.setPaidChg(paidChg);
			gcRiClaimXTty.setGenBillNo(null);
			gcRiClaimXTty.setCurrentVerInd(true);
			gcRiClaimXTty.setGrossIncurred(grossIncurred);
			gcRiClaimXTty.setGrossIncurredChg(grossIncurredChg);
			gcRiClaimXTty.setBillInd("0");
			gcRiClaimXTty.setRiCurrency(grXTtySectResVo.getCurrency());
			gcRiClaimXTty.setRiExchange(exrate);
			gcRiClaimXTty.setReportDate(new Date());
			gcRiClaimXTty.setTtyType(grXTtyLayer.getnPrpType());
			gcRiClaimXTty.setTransDate(gcRiClaimVo.getCreateTime());
			gcRiClaimXTty.setPreXolNo(preXolNo);
			gcRiClaimXTty.setPreTime(preTime);
			gcRiClaimXTty.setSaveFlag(isNotice);

			gcRiClaimXTty.setGrXTtyLayerResVo(grXTtyLayer);
			gcRiClaimXTty.setGrXTtySectResVo(grXTtySectResVo);
			gcRiClaimXTty.setDeductible(excessLoss);
			gcRiClaimXTty.setContQuota(contQuota);
			gcRiClaimXTty.setGrossOsReserve(mapAmount.get("grossOsReserve"));
			gcRiClaimXTty.setGrossPaid(mapAmount.get("grossPaid"));
			gcRiClaimXTty.setNetPaid(retenPaid);


			FieldUtils.setAllVoValue(gcRiClaimXTty, updateType, "system");
			gcRiClaimXTty.setCreateTime(gcRiClaimVo.getUpdateTime());
			this.ableXolRiskPrintData(gcRiClaimXTty, gcRiClaimVo);

			// ableXolRiskPrintData 基于 retentionRate 调整了 paid，这里据最终 paid 重算 shareRate 与 paidChg
			BigDecimal finalPaid = gcRiClaimXTty.getPaid();
			BigDecimal shareRate = retenPaid.compareTo(BigDecimal.ZERO) == 0 ? BigDecimal.ZERO : finalPaid.divide(retenPaid, 8, BigDecimal.ROUND_HALF_UP).multiply(BigDecimal.valueOf(100));
			gcRiClaimXTty.setShareRate(shareRate);
			BigDecimal finalPaidChg = (oldGcRiClaimXTty != null) ? finalPaid.subtract(oldPaidEffective == null ? BigDecimal.ZERO : oldPaidEffective) : finalPaid;
			if ("2".equals(updateType) && oldGcRiClaimXTty != null) {
				finalPaidChg = finalPaidChg.add(oldGcRiClaimXTty.getPaidChg());
			}
			gcRiClaimXTty.setPaidChg(finalPaidChg);
			gcRiClaimXTtyList.add(gcRiClaimXTty);
			if (b) {
				// 限额累计使用最终的 paid
				paid = finalPaid;
				if(null != oldGcRiClaimXTty && !"1".equals(oldGcRiClaimXTty.getBillInd())) {
					paid = paid.subtract(oldGcRiClaimXTty.getPaid());
				}
				// 限额更新，加赔付变化量
				map.put(grXTtySectResVo.getTtyId(), map.get(grXTtySectResVo.getTtyId()).add(paid));
			}
		}
	}
```

