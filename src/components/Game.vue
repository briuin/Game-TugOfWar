<template>
  <div :class="$style.game_area">
    <div v-if="resultMessage">
      <div :class="$style.modal_overlay">
        <div :class="$style.modal_container">
          <div :class="$style.modal_content">
            <div :class="$style.result_text">{{resultMessage}}</div>
            <button type="button" :class="$style.join_next" @click="nextRound">Join Next PK</button>
          </div>
        </div>
      </div>
    </div>
    <div :class="$style.hit_area">
      <div :class="$style.name">{{opponent.name}}</div>
      <div :class="$style.times">{{opponent.hitTimes}}</div>
    </div>
    <div :class="[$style.hit_area, $style.player]" @click="hit">
      <div :class="$style.times">{{player.hitTimes}}</div>
      <div :class="$style.name">{{player.name}}</div>
    </div>
    <h3>Click the blue area, 100 to Win!</h3>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import {Player} from '../models/player';
import {HubConnection, HubConnectionBuilder} from '@aspnet/signalr';
import {CONFIG} from '../environment';
import {HubAction} from '../enums/HubAction';

@Component
export default class HelloWorld extends Vue {
    @Prop() public playerDetail!: Player;
    public player: Player = new Player();
    public opponent: Player = new Player();
    public connection!: HubConnection;
    public maxHit = 100;
    public resultMessage = '';

    public hit() {
        if (this.isGameOver || this.isWaitingForOpponent) {
            return;
        }
        this.connection.invoke(HubAction.Hit);
        this.player.hitTimes++;
    }

    public created() {
        this.player = {...this.player, ...this.playerDetail};
        this.connection = new HubConnectionBuilder().withUrl(CONFIG.HUB_URL).build();

        this.connection.on('error', (data: string) => {
            alert(data);
        });

        this.connection.on('opponentJoined', (data: string) => {
            this.opponent.name = data;
        });

        this.connection.on('opponentLeave', () => {
            this.reset();
        });

        this.connection.on('opponentHit', (data: number) => {
            this.opponent.hitTimes = data;
        });

        this.connection.on('result', (data: {winner: string}) => {
            setTimeout(() => {
                this.resultMessage = `${data.winner} win!`;
            }, 500);
        });

        this.connection.start()
            .then(() => this.connection.invoke(HubAction.Join, this.player.name));
    }

    public nextRound() {
        this.reset();
        this.connection.invoke(HubAction.NextRound, this.player.name);
    }

    private reset() {
        this.player.hitTimes = 0;
        this.opponent = new Player();
        this.resultMessage = '';
    }

    private get isGameOver() {
        return this.player.hitTimes >= this.maxHit || this.opponent.hitTimes >= this.maxHit;
    }

    private get isWaitingForOpponent() {
        return this.opponent.name === '';
    }
}
</script>

<style lang="scss" module>
  .game_area {
    max-width: 300px;
    width: 100%;
    display: flex;
    justify-content: center;
    align-content: center;
    flex-direction: column;
    margin: 0 auto;
    text-align: center;
  }

  .hit_area {
    height: 30vh;
    display: flex;
    background: bisque;
    border: 1px solid black;
    flex-direction: column;
  }

  .player {
    background: aquamarine;
  }

  .name {
    height: 10%;
    text-align: center;
  }

  .times {
    height: 90%;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 3em;
  }

  .modal {
    &_overlay {
      position: fixed;
      width: 100%;
      height: 100vh;
      background: rgba(0, 0, 0, 0.9);
      left: 0;
      top: 0;
      z-index: 1000;
    }

    &_container {
      position: absolute;
      top: 50%;
      -ms-transform: translateY(-50%);
      transform: translateY(-50%);
      left: 0;
      right: 0;
    }

    &_content {
      display: flex;
      flex-direction: column;
      width: 300px;
      margin: 0 auto;
    }
  }

  .join_next {
    height: 35px;
    font-size: 1.2em;
    background: aquamarine;
    cursor: pointer;
    color: darkslategray;
  }

  .result_text {
    font-size: 2em;
    text-align: center;
    margin-bottom: 50px;
    color: aquamarine;
  }
</style>
