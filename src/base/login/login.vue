<template>
	<div class="form-common">
		<mt-header class="top" fixed title="登录页">
       <router-link to="/" slot="left">
    <mt-button icon="back">返回</mt-button>
  </router-link>
		</mt-header>
		<form @submit.prevent="login">
			<div class="formc-cnt">
				<span>账号：</span>
				<input auto-focus placeholder="手机/邮箱" type="text" required v-model="name" maxlength="20" />
			</div>
			<div class="formc-cnt">
				<span>密码：</span>
				<input type="password" v-model="pwd" required maxlength="20" />
			</div>
			<div>
				<mt-button id="loginBtn" type="primary" v-bind:class="{'is-disabled':(!name||!pwd||loading)}" size="large">登录</mt-button>
        <p class="register" @click="register">还没账号？注册</p>
			</div>
		</form>
	</div>
</template>

<script>
export default {
  name: "login",
  data() {
    return {
      name: "",
      pwd: "",
      loading: false
    };
  },
  components: {},
  methods: {
    login() {
      console.log("hihihihihi");
    },
    register(){
      console.log('你好呀，欢迎注册');
    }
  }
};
</script>

<style scoped>
.form-common {
  position: fixed;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  z-index: 998;
  background: #eeeeee;
}

.top{
  font-size: 16px;
}

.form-common {
  padding: 5em 5% 0;
}

.formc-cnt {
  display: flex;
  border-bottom: 1px solid #eee;
  align-items: center;
  padding: 0.5em 0;
}

#loginBtn {
  margin-top: 1.2em;
}

.formc-cnt span {
  text-align: left;
}

.formc-cnt input {
  flex: 1;
  display: block;
  background: transparent;
  padding: 0.5em 0.5em;
  border-bottom: 1px solid lightskyblue;
}

.register {
  display: flex;
  justify-content: center;
  margin: 20px;
}
</style>