<template>
	<div class="plan">
		<div class="justify-content-between mb-2 d-flex justify-content-between">
			<div class="text-start">
				<div class="label">{{ $t("main.targetChargePlan.chargeDuration") }}</div>
				<div
					:class="`value  d-sm-flex align-items-baseline ${
						timeWarning ? 'text-warning' : 'text-primary'
					}`"
				>
					<div>{{ planDuration }}</div>
					<div v-if="fmtPower" class="extraValue text-nowrap ms-sm-1">
						{{ fmtPower }}
					</div>
				</div>
			</div>
			<div v-if="hasTariff" class="text-end">
				<div class="label">
					<span v-if="activeSlot">{{ activeSlotName }}</span>
					<span v-else-if="isCo2">{{ $t("main.targetChargePlan.co2Label") }}</span>
					<span v-else>{{ $t("main.targetChargePlan.priceLabel") }}</span>
				</div>
				<div class="value text-primary">
					{{ fmtAvgPrice }}
				</div>
			</div>
		</div>
		<TariffChart
			:slots="slots"
			:target-text="targetText"
			:target-offset="targetHourOffset"
			@slot-hovered="slotHovered"
		/>
	</div>
</template>

<script>
import formatter from "../../mixins/formatter.js";
import { CO2_TYPE } from "../../units.js";
import TariffChart from "../Tariff/TariffChart.vue";

export default {
	name: "ChargingPlanPreview",
	components: { TariffChart },
	mixins: [formatter],
	props: {
		duration: Number,
		power: Number,
		rates: Array,
		plan: Array,
		smartCostType: String,
		targetTime: Date,
		currency: String,
	},
	data() {
		return { activeIndex: null, startTime: new Date() };
	},
	computed: {
		endTime() {
			if (!this.plan?.length) {
				return null;
			}
			const end = this.plan[this.plan.length - 1].end;
			return end ? new Date(end) : null;
		},
		timeWarning() {
			if (this.targetTime && this.endTime) {
				return this.targetTime < this.endTime;
			}
			return false;
		},
		planDuration() {
			return this.fmtDuration(this.duration);
		},
		fmtPower() {
			if (this.duration > 0 && this.power > 0) {
				return `@ ${this.fmtW(this.power)}`;
			}
			return null;
		},
		isCo2() {
			return this.smartCostType === CO2_TYPE;
		},
		hasTariff() {
			return this.rates?.length > 1;
		},
		avgPrice() {
			let hourSum = 0;
			let priceSum = 0;
			this.convertDates(this.plan).forEach((slot) => {
				const hours = (slot.end.getTime() - slot.start.getTime()) / 3600000;
				if (slot.price) {
					hourSum += hours;
					priceSum += hours * slot.price;
				}
			});
			return hourSum ? priceSum / hourSum : undefined;
		},
		fmtAvgPrice() {
			if (this.duration === 0) {
				return "—";
			}
			const price = this.activeSlot ? this.activeSlot.price : this.avgPrice;
			if (price === undefined) {
				return this.$t("main.targetChargePlan.unknownPrice");
			}
			return this.isCo2
				? this.fmtCo2Medium(price)
				: this.fmtPricePerKWh(price, this.currency);
		},
		activeSlot() {
			return this.slots[this.activeIndex];
		},
		activeSlotName() {
			if (this.activeSlot) {
				const { day, startHour, endHour } = this.activeSlot;
				const range = `${startHour}–${endHour}`;
				return this.$t("main.targetChargePlan.timeRange", { day, range });
			}
			return null;
		},
		targetHourOffset() {
			if (!this.targetTime) {
				return null;
			}
			const start = new Date(this.startTime);
			start.setMinutes(0);
			start.setSeconds(0);
			start.setMilliseconds(0);
			return (this.targetTime.getTime() - start.getTime()) / (60 * 60 * 1000);
		},
		targetText() {
			if (!this.targetTime) {
				return null;
			}
			return this.fmtWeekdayTime(this.targetTime);
		},
		slots() {
			const result = [];
			const rates = this.convertDates(this.rates);
			const plan = this.convertDates(this.plan);
			const oneHour = 60 * 60 * 1000;
			for (let i = 0; i < 39; i++) {
				const start = new Date(this.startTime.getTime() + oneHour * i);
				const startHour = start.getHours();
				start.setMinutes(0);
				start.setSeconds(0);
				start.setMilliseconds(0);
				const end = new Date(start.getTime());
				end.setHours(startHour + 1);
				const endHour = end.getHours();
				const day = this.weekdayShort(start);
				const toLate = this.targetTime && this.targetTime <= start;
				// TODO: handle multiple matching time slots
				const price = this.findSlotInRange(start, end, rates)?.price;
				const isTarget = start <= this.targetTime && end > this.targetTime;
				const charging = this.findSlotInRange(start, end, plan) != null;
				const warning =
					charging &&
					this.targetTime &&
					end > this.targetTime &&
					this.targetTime < this.endTime;
				result.push({
					day,
					price,
					startHour,
					endHour,
					charging,
					toLate,
					warning,
					isTarget,
				});
			}
			return result;
		},
	},
	watch: {
		rates() {
			this.startTime = new Date();
		},
	},
	methods: {
		convertDates(list) {
			if (!list?.length) {
				return [];
			}
			return list.map((item) => {
				return {
					start: new Date(item.start),
					end: new Date(item.end),
					price: item.price,
				};
			});
		},
		findSlotInRange(start, end, slots) {
			return slots.find((s) => {
				if (s.start.getTime() < start.getTime()) {
					return s.end.getTime() > start.getTime();
				}
				return s.start.getTime() < end.getTime();
			});
		},
		slotHovered(index) {
			this.activeIndex = index;
		},
	},
};
</script>

<style scoped>
.value {
	font-size: 18px;
	font-weight: bold;
}
.extraValue {
	color: var(--evcc-gray);
	font-size: 14px;
}
.label {
	color: var(--evcc-gray);
	text-transform: uppercase;
}
</style>
