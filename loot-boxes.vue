<template>
  <div
    class="smooth-div"
    @mouseover="onHover"
    @mouseleave="onMouseLeave"
    :style="rootStyle"
  >
    <button
      style="margin-top: 10px"
      v-if="feed && lootcountnoskin >= 3"
      :disabled="user_state.local.revealing || claimingall || user_state.local.spinning"
      v-on:click.exact="onFeed(Math.min(lootcountnoskin, 10))"
      v-on:click.ctrl.alt.shift.exact="onClaimAll()"
      class="btn btn-light"
    >
      Feed {{ Math.min(lootcountnoskin, 10) }}
    </button>
    <button
      style="margin-left: 10px; margin-top: 10px"
      v-if="feed && lootcountnoskin >= 30"
      :disabled="user_state.local.revealing || claimingall || user_state.local.spinning"
      v-on:click.exact="onFeed(Math.min(lootcountnoskin, 60))"
      v-on:click.ctrl.alt.shift.exact="onClaimAll()"
      class="btn btn-light"
    >
      Feed {{ Math.min(lootcountnoskin, 60) }}
    </button>
    <div
      style="display: flex; justify-content: flex-end"
      :style="{ width: 30 * user_state.local.boxes_per_row + 'px' }"
    >
      <div style="display: flex; color: white; align-items: center; margin-top: 5px">
        <transition-group
          name="list-complete"
          tag="div"
          style="
            display: flex;
            align-items: center;
            flex-wrap: wrap;
            width: 100%;
            justify-content: flex-end;
          "
        >
          <div
            :key="key + 1"
            v-for="(value, key) in lootnoskin"
            :class="lootClass(key)"
            :title="
              value[0][0] == '.' && value[0].length > 1
                ? Math.round((value[1] / value[2]) * 100) / 10000 +
                  ' ROY for killing ' +
                  value[0].slice(1)
                : Math.round((value[1] / value[2]) * 100) / 10000 + ' ROY'
            "
            :style="[baseStyle, lootStyle(value[0])]"
            v-on:click.exact="onClick(key)"
            v-on:click.ctrl.alt.shift.exact="onClaimAll()"
          >
            <span class="box-content" style="border: white solid" v-if="value[3]">{{
              [...value[3]][0]
            }}</span>
            <!-- <span class="box-content" v-else-if="value[0] == 'T'">â­</span> -->
            <span
              class="box-content"
              style="
                border: white solid;
                color: white;
                font-weight: 600;
                vertical-align: middle;
                line-height: 20px;
              "
              v-else-if="value[0][0] == '.'"
              >{{ value[0][1] || "&nbsp;" }}</span
            >
          </div>

          <div
            :key="key + 1"
            v-for="(value, key) in lootgrouped"
            :style="[baseStyle, lootStyle(key)]"
            :title="Object.keys(value).length + ' random ' + key.slice(1) + ' skin(s)'"
            class="prevent-boost cardList list-complete-item"
            v-on:click.exact="onClickSkin(key)"
          >
            <!-- <span class="box-content" v-else-if="value[0] == 'T'">â­</span> -->
            <span
              class="box-content"
              style="
                color: black;
                font-weight: 600;
                vertical-align: middle;
                font-size: 17px;
                line-height: 24px;
              "
              >{{ Object.keys(value).length > 1 ? Object.keys(value).length : "" }}</span
            >
          </div>
        </transition-group>
      </div>
    </div>
    <span
      v-if="prize"
      key="s2"
      class="glowing-good"
      style="margin-right: 5px; margin-top: 10px; font-weight: bold; padding: 4px"
      >{{ "+" + prize + " " + user_state.cloud.currency || "ROY" + "!" }}</span
    >
    <span
      v-if="message"
      key="s3"
      class="glowing-good"
      style="margin-right: 5px; margin-top: 10px; font-weight: bold; padding: 4px"
      >{{ message }}</span
    >
    <!-- <button
      style="margin-top: 10px"
      v-if="hovering && lootcount > 5 && !feed"
      onclick="window.open('/wheel.html?popup=true', 
                         'newwindow', 
                         'width=1200,height=850,left=200, top=200');  return false;"
      class="btn btn-light glowi"
    >
      Spin The Wheel
    </button>
   -->
  </div>
</template>

<script>
module.exports = {
  props: {
    buttontext: {
      type: String,
      required: false,
    },
    feed: {
      type: Boolean,
      required: false,
    },
  },
  setup(props) {
    const { onMounted, ref, watch, computed, reactive } = Vue;
    const hovering = ref(false);
    const message = ref("");
    const prize = ref(0);
    const claimingall = ref(false);
    let loot = ref({});
    let glowing = ref({});
    let shaking = ref({});
    const kaching = props.feed
      ? new Audio("/assets/sounds/arcade_coin.mp3")
      : new Audio("/assets/sounds/kaching.mp3");
    kaching.volume = 0.3;

    onMounted(() => {
      loot.value = reactive(Object.assign({}, user_state.cloud.loot));
      glowing.value = {};
      shaking.value = {};
    });

    watch(user_state.cloud.loot, (count, prevCount) => {
      if (!user_state.local.revealing) {
        loot.value = reactive(Object.assign({}, user_state.cloud.loot));
        const loots = Object.keys(loot.value);
        if (loots.length > user_state.local.max_boxes) {
          let maxclaim = loots.length - user_state.local.max_boxes;
          let claimed = 0;
          for (let i = loots.length - 1; i >= 0 && claimed < maxclaim; i--) {
            if (loots[i][0] != "-" && loot.value[loots[i]][0][0] != "?") {
              claimed += 1;
              setTimeout(() => onClick(loots[i], true), 1000 * claimed);
              // return;
            }
          }
        }
      }
    });

    const lootcountnoskin = computed(() => {
      return Object.values(loot.value).filter((v) => v[0][0] != "?").length;
      // return user_state.local.revealing ? loot : user_state.cloud.loot;
    });

    const groupBy = (items, key) =>
      items.reduce(
        (result, item) => ({
          ...result,
          [item[key]]: [...(result[item[key]] || []), item],
        }),
        {}
      );

    const lootgrouped = computed(() => {
      return groupBy(
        Object.values(loot.value).filter((v) => v[0][0] == "?"),
        0
      );
    });

    const lootnoskin = computed(() => {
      return Object.fromEntries(
        Object.entries(loot.value).filter((v) => v[1][0][0] != "?")
      );
    });

    const lootcount = computed(() => {
      return Object.values(loot.value).length;
      // return user_state.local.revealing ? loot : user_state.cloud.loot;
    });

    const onClaimAll = () => {
      if (claimingall.value || !feedable) {
        return;
      } else {
        claimingall.value = true;
        _onClaimAll();
      }
    };

    const _onClaimAll = () => {
      let lootkeys = Object.entries(loot.value)
        .filter((kv) => kv[1][0][0] != "?")
        .map((kv) => kv[0]);
      //Object.keys(loot.value);

      if (lootkeys.length > 1) {
        onClick(lootkeys[0], false);
        setTimeout(() => _onClaimAll(), props.feed ? 333 : 500);
      } else if (lootkeys.length == 1) {
        onClick(lootkeys[0], false);
        claimingall.value = false;
      }
    };

    window.claimAllBoxes = onClaimAll;

    const onFeed = (amount, nosound) => {
      if (!feedable) {
        return;
      }
      let lootkeys = Object.entries(loot.value)
        .filter((kv) => kv[1][0][0] != "?")
        .map((kv) => kv[0])
        .slice(0, amount);

      if (lootkeys.length < amount) {
        return;
      }

      lootkeys.forEach((l) => (shaking.value[l] = true));
      user_state.local.walletsounds = false;

      const response = (ack) => {
        if (!ack) {
          user_state.local.walletsounds = true;
          return;
        }
        user_state.local.revealing += amount;
        user_state.local.feed_result = ack;

        setTimeout(() => {
          if (!nosound) {
            kaching.pause();
            kaching.currentTime = 0;
            kaching.play();
          }
          lootkeys.forEach((l) => delete loot.value[l]);
          setTimeout(() => {
            user_state.local.revealing -= amount;
            if (user_state.local.revealing <= 0) {
              user_state.local.walletsounds = true;
              glowing.value = {};
            }
          }, 2000);
        }, 200);
      };

      coordinator.emit("feed10", lootkeys, response);
    };

    const onClickSkin = (rarity) => {
      const skinbox = Object.entries(loot.value).find((kv) => kv[1][0] == rarity);
      if (skinbox) {
        onClick(skinbox[0]);
      }
    };

    const onClick = (key, nosound) => {
      if (!feedable || (props.feed && loot.value[key] && loot.value[key][0][0] == "?")) {
        return;
      }
      shaking.value[key] = true;
      user_state.local.walletsounds = false;

      const response = (ack) => {
        if (!ack || ack.notopened) {
          if (ack && ack.notopened && !props.feed) {
            message.value = ack.message;
            setTimeout(() => {
              message.value = "";
            }, 2000);
          }
          user_state.local.walletsounds = true;
          return;
        }
        user_state.local.revealing += 1;

        if (!props.feed) {
          message.value = ack.message;
          if (key in loot.value) {
            loot.value[key][0] = ack.revealed;
          }
        } else {
          user_state.local.feed_result = ack;
        }
        glowing.value[key] = 1;

        setTimeout(() => {
          if (!props.feed) {
            prize.value = Math.round(strip(prize.value + ack.prize) * 100) / 100;
          }
          if (!nosound) {
            kaching.pause();
            kaching.currentTime = 0;
            if (!props.feed) {
              kaching.volume = (user_state.local.volume / 100) * 0.15;
            }
            kaching.play();
          }
          delete loot.value[key];
          // Vue.delete(loot.value, key);
          setTimeout(() => {
            user_state.local.revealing -= 1;
            if (user_state.local.revealing <= 0) {
              user_state.local.walletsounds = true;
              message.value = "";
              prize.value = 0;
              glowing.value = {};
            }
          }, 2000);
        }, 200);
      };
      if (!props.feed) {
        socket.emit("reveal", key || -1, response);
      } else {
        coordinator.emit("feed", key || -1, response);
      }
    };

    const feedable = computed(() => {
      return !(
        user_state.local.spinning ||
        (user_state.local.feed_result &&
          user_state.local.feed_result.wallet &&
          user_state.local.feed_result.wallet.wheel_total > 1000)
      );
    });

    const baseStyle = computed(() => {
      return {
        "background-color": "white",

        width: "24px",
        height: "24px",
      };
    });

    const rootStyle = computed(() => {
      if (lootcount.value > 0 || prize.value) {
        return {
          height:
            Math.ceil(lootcount.value / user_state.local.boxes_per_row) * 29 +
            (props.feed ? 50 : 0) +
            "px",
        };
      } else {
        return {
          height: "0px",
        };
      }
    });

    const lootClass = (key) => {
      return {
        "prevent-boost": true,
        cardList: true,
        "list-complete-item": true,

        "glowing-good": glowing.value[key] === 1,
        "glowing-bad": glowing.value[key] === 2,
        // shaking: !glowing.value[key] && shaking.value[key],
      };
    };

    const skinClass = (key) => {
      return {
        "prevent-boost": true,
        cardList: true,
        "list-complete-item": true,
        // shaking: !glowing.value[key] && shaking.value[key],
      };
    };

    const onHover = () => {
      hovering.value = true;
    };

    const onMouseLeave = () => {
      hovering.value = false;
    };

    const lootStyle = (color) => {
      // let color = loot[index] && loot[index].v;
      const new_design = user_state.local.new_design;
      let style = {};
      if (!color) {
        color = "#FFFFFF"; //new_design ? "#5D6EFF" : "#FFFFFF";
      } else if (color == "R") {
        color = user_state.local.colors.pastelred.toHexString();
      } else if (color == "S") {
        color = user_state.local.colors.pastelgreen.toHexString();
      } else if (color == "P") {
        color = user_state.local.colors.pastelblue.toHexString();
      } else if (color == "T") {
        color = new_design ? "#ffffff" : user_state.local.colors.neonyellow.toHexString();
      } else if (color[0] == ".") {
        color = "#1e1542";
      } else if (color[0] == "?") {
        const rarity = color.slice(1);
        if (rarity == "legendary") {
          color = "#ffd700";
        } else if (rarity == "epic") {
          color = "#df00fe";
        } else if (rarity == "rare") {
          color = "#68F365";
        } else if (rarity == "uncommon") {
          color = "#cd7f32";
        } else if (rarity == "common") {
          color = "white";
        } else {
          color = "white";
        }
        style["border-radius"] = "12px";
      }

      style["background-color"] = color;
      return style;
    };

    return {
      user_state,
      game_state,
      onClick,
      baseStyle,
      lootStyle,
      lootClass,
      skinClass,
      message,
      loot,
      lootcount,
      lootcountnoskin,
      glowing,
      prize,
      onHover,
      onMouseLeave,
      hovering,
      onClaimAll,
      rootStyle,
      onFeed,
      lootgrouped,
      lootnoskin,
      onClickSkin,
    };
  },
};
</script>

<style scoped>
.box-content {
  display: block;
  text-align: center;
  line-height: 18px;
  color: black;
  cursor: pointer;
}
.smooth-div {
  transition: 0.75s;
}
.list-complete-item {
  transition: all 0.25s;
  display: inline-block;
  margin: 3px;
}

.list-complete-item:hover {
  transform: scale(1.2);
  /* border: black solid 2px !important; */
  /* transition: scale 0.3s; */
}
.list-complete-enter
/* .list-complete-leave-active below version 2.1.8 */ {
  /* transition: all 1s; */
  opacity: 0;
  transform: translateX(+30px);
}

.list-complete-leave-to {
  opacity: 0;
  transform: translateY(+30px);
}
.list-complete-leave-active {
  position: absolute;
}

.glowing-good {
  animation: glow-good 0.2s infinite alternate;
}

.shaking {
  animation: shake 0.5s cubic-bezier(0.36, 0.07, 0.19, 0.97) both;
}

.glowing-text {
  /* animation: glow-text 1s ease-in-out infinite alternate; */
}

@keyframes glow-text {
  from {
    text-shadow: 0 0 10px #fff, 0 0 20px #fff, 0 0 30px #e60073, 0 0 40px #e60073,
      0 0 50px #e60073, 0 0 60px #e60073, 0 0 70px #e60073;
  }
  to {
    text-shadow: 0 0 20px #fff, 0 0 30px #ff4da6, 0 0 40px #ff4da6, 0 0 50px #ff4da6,
      0 0 60px #ff4da6, 0 0 70px #ff4da6, 0 0 80px #ff4da6;
  }
}

.glowing-bad {
  animation: glow-bad 0.6s infinite alternate;
}

@keyframes glow-good {
  from {
    box-shadow: 0 0 2px -2px white;
  }
  to {
    box-shadow: 0 0 2px 2px white;
  }
}

@keyframes glow-bad {
  from {
    box-shadow: 0 0 5px -5px black;
  }
  to {
    box-shadow: 0 0 5px 5px black;
  }
}

@keyframes shake {
  10%,
  90% {
    transform: translate3d(-1px, -1px, 0);
  }

  20%,
  80% {
    transform: translate3d(2px, 2px, 0);
  }

  30%,
  50%,
  70% {
    transform: translate3d(-3px, -3px, 0);
  }

  40%,
  60% {
    transform: translate3d(3px, 3px, 0);
  }
}
</style>
