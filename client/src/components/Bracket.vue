
<template>
	<div v-if="bracket">
		<div v-if="displayControls" class="controls">
			<button @click="copyLink" v-tooltip="'Copy link to a read-only version of this bracket to the clipboard.'">
				<i class="fas fa-clipboard"></i> Copy Link to Clipboard
			</button>
			<div v-if="fullcontrol">
				<span v-tooltip="'If set, only the owner will be able to enter results.'">
					<input type="checkbox" id="lock" :checked="locked" @change="lock($event)" />
					<label for="lock"> <i class="fas fa-lock"></i> Lock </label>
				</span>
				<button @click="$emit('generate')">
					{{ !teamDraft ? "Re-Generate Single Elimination" : "Re-Generate Team Bracket" }}
				</button>
				<button v-if="!teamDraft" @click="$emit('generate-swiss')">Re-Generate 3-Round Swiss</button>
			</div>
			<div v-else-if="locked">
				<span> <i class="fas fa-lock"></i> Bracket is locked. Only the Session Owner can enter results. </span>
			</div>
		</div>
		<div class="bracket-columns">
			<div class="bracket-column" v-for="(col, colIndex) in matches" :key="colIndex">
				<div v-for="(m, matchIndex) in col" :key="matchIndex" class="bracket-match">
					<td class="bracket-match-num">{{ m.index + 1 }}</td>
					<td class="bracket-match-players">
						<div v-for="(p, index) in m.players" :key="index">
							<div class="bracket-player bracket-empty" v-if="p.empty">(Empty)</div>
							<div class="bracket-player bracket-tbd" v-else-if="p.tbd">(TBD {{ p.tbd }})</div>
							<div
								class="bracket-player"
								:class="{
									'bracket-winner':
										bracket.results[m.index][index] > bracket.results[m.index][(index + 1) % 2],
									teama: bracket.teamDraft && index % 2 === 0,
									teamb: bracket.teamDraft && index % 2 === 1,
								}"
								v-else
							>
								<template v-if="colIndex === 2">
									<i v-if="isGold(p, index)" class="trophy gold fas fa-trophy"></i>
									<i v-else-if="isSilver(p)" class="trophy silver fas fa-trophy"></i>
									<div v-else class="trophy"></div>
								</template>
								<div
									class="bracket-player-name"
									v-tooltip="'Current record: ' + recordString(p)"
									:class="{ clickable: draftlog }"
									@click="if (draftlog) selectedUser = p;"
								>
									{{ p.userName }}
									<i
										class="fas fa-clipboard-check green"
										v-if="hasDeckList(p.userID)"
										v-tooltip="`${p.userName} submited their deck. Click to review it.`"
									></i>
								</div>
								<template v-if="m.isValid()">
									<input
										v-if="editable"
										class="result-input"
										type="number"
										v-model.number="bracket.results[m.index][index]"
										min="0"
										@change="emitUpdated"
									/>
									<div class="bracket-result" v-else>{{ bracket.results[m.index][index] }}</div>
								</template>
							</div>
						</div>
					</td>
				</div>
			</div>
		</div>
		<div v-if="draftlog && selectedUser">
			<h1>{{ selectedUser.userName }}'s deck</h1>
			<decklist
				:list="selectedDeckList"
				:carddata="draftlog.carddata"
				:username="selectedUser.userName"
				:language="language"
			/>
		</div>
	</div>
	<div v-else>No valid bracket.</div>
</template>

<script>
import { copyToClipboard } from "../helper";
import { fireToast } from "../alerts";
import Decklist from "./Decklist.vue";

export default {
	name: "Bracket",
	components: { Decklist },
	data: () => {
		return {
			selectedUser: null,
		};
	},
	props: {
		bracket: { type: Object, required: true },
		teamDraft: { type: Boolean, required: false },
		displayControls: { type: Boolean, default: true },
		editable: { type: Boolean, default: false },
		locked: { type: Boolean, default: false },
		fullcontrol: { type: Boolean, default: false },
		sessionID: { type: String },
		draftlog: { type: Object, default: null },
		language: { type: String, required: true },
	},
	methods: {
		teamWins: function (team) {
			let total = 0;
			for (let col of this.matches) {
				for (let m of col) {
					if (m.isValid() && this.bracket.results[m.index][0] !== this.bracket.results[m.index][1]) {
						let winIdx = this.bracket.results[m.index][0] > this.bracket.results[m.index][1] ? 0 : 1;
						if (winIdx === team) {
							total += 1;
						}
					}
				}
			}
			return total;
		},
		isGold: function (p, index) {
			if (this.bracket.teamDraft) {
				return this.teamWins(index) >= 5;
			} else {
				return this.records[p.userID].wins === 3;
			}
		},
		isSilver: function (p) {
			return !this.bracket.teamDraft && this.records[p.userID].wins === 2;
		},
		emitUpdated: function () {
			this.$emit("updated");
		},
		recordString: function (p) {
			return `${this.records[p.userID].wins} - ${this.records[p.userID].losses}`;
		},
		lock: function (e) {
			this.$emit("lock", e.target.checked);
		},
		copyLink: function () {
			copyToClipboard(
				`${window.location.protocol}//${window.location.hostname}${
					window.location.port ? ":" + window.location.port : ""
				}/bracket?session=${encodeURI(this.sessionID)}`
			);
			fireToast("success", "Bracket Link copied to clipboard!");
		},
		hasDeckList: function (userID) {
			return this.draftlog && this.draftlog.users[userID] && this.draftlog.users[userID].decklist;
		},
	},
	computed: {
		matches: function () {
			let m = [[], [], []];
			const Match = function (index, players) {
				this.index = index;
				this.players = players;
				this.isValid = function () {
					return (
						!this.players[0].empty && !this.players[1].empty && !this.players[0].tbd && !this.players[1].tbd
					);
				};
			};

			const winner = (match) => {
				if (match.players[0].empty && match.players[1].empty) return { empty: true };
				if (match.players[0].empty) return match.players[1];
				if (match.players[1].empty) return match.players[0];
				if (
					!this.bracket.results ||
					this.bracket.results[match.index][0] === this.bracket.results[match.index][1]
				)
					return { tbd: "W" + (match.index + 1) };
				if (this.bracket.results[match.index][0] > this.bracket.results[match.index][1])
					return match.players[0];
				else return match.players[1];
			};

			const loser = (match) => {
				if (match.players[0].empty || match.players[1].empty) return { empty: true };
				if (
					!this.bracket.results ||
					this.bracket.results[match.index][0] === this.bracket.results[match.index][1]
				)
					return { tbd: "L" + (match.index + 1) };
				if (this.bracket.results[match.index][0] > this.bracket.results[match.index][1])
					return match.players[1];
				else return match.players[0];
			};

			const playerOrEmpty = (idx) => {
				return this.bracket.players[idx] ? this.bracket.players[idx] : { empty: true };
			};

			if (this.bracket.teamDraft) {
				m[0].push(new Match(0, [playerOrEmpty(0), playerOrEmpty(3)]));
				m[0].push(new Match(1, [playerOrEmpty(2), playerOrEmpty(5)]));
				m[0].push(new Match(2, [playerOrEmpty(4), playerOrEmpty(1)]));
				m[1].push(new Match(3, [playerOrEmpty(0), playerOrEmpty(5)]));
				m[1].push(new Match(4, [playerOrEmpty(2), playerOrEmpty(1)]));
				m[1].push(new Match(5, [playerOrEmpty(4), playerOrEmpty(3)]));
				m[2].push(new Match(6, [playerOrEmpty(0), playerOrEmpty(1)]));
				m[2].push(new Match(7, [playerOrEmpty(2), playerOrEmpty(3)]));
				m[2].push(new Match(8, [playerOrEmpty(4), playerOrEmpty(5)]));
				return m;
			}

			for (let i = 0; i < 4; ++i) {
				m[0].push(new Match(i, [playerOrEmpty(2 * i), playerOrEmpty(2 * i + 1)]));
			}
			m[1].push(new Match(4, [winner(m[0][0]), winner(m[0][1])]));
			m[1].push(new Match(5, [winner(m[0][2]), winner(m[0][3])]));
			if (this.bracket.swiss) {
				m[1].push(new Match(6, [loser(m[0][2]), loser(m[0][3])]));
				m[1].push(new Match(7, [loser(m[0][0]), loser(m[0][1])]));
				m[2].push(new Match(8, [winner(m[1][0]), winner(m[1][1])]));
				m[2].push(new Match(9, [loser(m[1][0]), winner(m[1][2])]));
				m[2].push(new Match(10, [loser(m[1][1]), winner(m[1][3])]));
				m[2].push(new Match(11, [loser(m[1][2]), loser(m[1][3])]));
			} else {
				m[2].push(new Match(6, [winner(m[1][0]), winner(m[1][1])]));
			}
			return m;
		},
		records: function () {
			let r = {};
			for (let p of this.bracket.players) if (p) r[p.userID] = { wins: 0, losses: 0 };
			for (let col of this.matches)
				for (let m of col) {
					if (m.isValid() && this.bracket.results[m.index][0] !== this.bracket.results[m.index][1]) {
						let winIdx = this.bracket.results[m.index][0] > this.bracket.results[m.index][1] ? 0 : 1;
						r[m.players[winIdx].userID].wins += 1;
						r[m.players[(winIdx + 1) % 2].userID].losses += 1;
					} else if (m.players[1].empty && !m.players[0].empty && !m.players[0].tbd) {
						r[m.players[0].userID].wins += 1;
					} else if (m.players[0].empty && !m.players[1].empty && !m.players[1].tbd) {
						r[m.players[1].userID].wins += 1;
					}
				}
			return r;
		},
		selectedDeckList: function () {
			if (this.draftlog.users[this.selectedUser.userID])
				return this.draftlog.users[this.selectedUser.userID].decklist;
			return null;
		},
	},
};
</script>

<style scoped>
.controls {
	padding: 0.5em;
}

.bracket-columns {
	display: flex;
}

.bracket-column {
	display: flex;
	flex-direction: column;
	justify-content: space-around;
}

.bracket-match {
	margin: 0.5em;
}

.bracket-match-num {
	vertical-align: middle;
	min-width: 1.5em;
}

.bracket-match-players {
	background: #333;
	border-radius: 1em;
	padding: 0.5em;
}

.bracket-player {
	display: flex;
	background: #2c2c2c;
	justify-content: space-between;
	height: 2em;
	line-height: 2em;
	width: 20rem;
	padding: 0.5em;
	margin: 0.5em;
	border-radius: 8px;
}

.bracket-winner {
	font-weight: bold;
	box-shadow: 0 0 4px 4px #bbb;
}

.bracket-tbd,
.bracket-empty {
	color: grey;
	pointer-events: none;
	user-select: none;
}

.bracket-result {
	font-size: 2em;
	min-width: 32px;
	text-align: right;
}

.bracket-player-name {
	font-size: 1.5em;
	max-width: 15rem;
	overflow: hidden;
}

.trophy {
	height: 32px;
	width: 32px;
	font-size: 32px;
}

.gold {
	color: gold;
}

.silver {
	color: silver;
}

.result-input {
	width: 2.2em;
}
</style>
