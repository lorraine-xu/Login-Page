<template xmlns:v-slot="http://www.w3.org/1999/XSL/Transform">
  <div class="login-form-wrap">
    <div class='login-in' style="display:none">
      {{ $t('signIn.title') }}
    </div>
    <div class='login-msg' style="display:none">
      {{ $t('signIn.subtitle') }}
    </div>
    <v-form ref="form" v-model="loginForm.valid" style="margin-top: 20px;display:none"
            lazy-validation>
      <v-text-field v-model="loginForm.userEmail"
                    :rules="[...emailRules, ...fieldRequiredRules(this.$t('user.email'))]"
                    :label="$t('user.email')" required outline @change="loadVerifyCodeImg"
                    v-on:keyup.enter="login"></v-text-field>
      <v-text-field type="password" v-model="loginForm.userPw" :label="$t('user.password')"
                    :rules="passwordRules"
                    required outline v-on:keyup.enter="login"></v-text-field>
      <v-layout wrap v-if="emailValid">
        <v-flex xs7>
          <v-text-field v-model="loginForm.verifyCode" required outline
                        v-on:keyup.enter="login"></v-text-field>
        </v-flex>
        <v-flex xs5>
          <img :src="verifyImage" alt="" v-if="verifyImage" @click="loadVerifyCodeImg">
        </v-flex>
      </v-layout>
    </v-form>

    <img class="decoration" referrerpolicy="no-referrer" src="../assets/decoration.png" alt=""/>

    <div class="buttons">
      <div v-if="logining == false">
        <v-btn large block color="white" style="height: 50px;" class="login-button-1"
               @click="ssoLogin" v-on:keyup.enter="ssoLogin">
          {{ $t('signIn.supplierLogin') }}
        </v-btn>
      </div>
      <div v-if="logining == true">
        <v-btn large block color="white" style="height: 50px;" class="login-button-1">
          {{ $t('signIn.supplierLogin') }}
        </v-btn>
      </div>
      <v-btn large block color="primary" style="height: 50px;" class="login-button-2"
             @click="adfsLogin" v-on:keyup.enter="adfsLogin">
        {{ $t('signIn.boschEmployeeLogin') }}
      </v-btn>
      <div class="image-text">
        <v-menu offset-y>
          <template v-slot:activator="{ on }">
            <v-btn text class="pl-2 pr-2 language-button" v-on="on">
              <span class="text-capitalize">{{ selectedLang }}</span>
              <v-icon style="margin: 0 0 0 8px">iconfont icon-down</v-icon>
            </v-btn>
          </template>
          <v-list class="user-setting-list">
            <v-list-item-group>
              <v-list-item v-for="language in languages" :key="language.value"
                           :value="language.value"
                           @click="onUpdateLang(language)">
                <v-list-item-title>{{ language.displayTxt }}</v-list-item-title>
              </v-list-item>
            </v-list-item-group>
          </v-list>
        </v-menu>
      </div>
      <img class="logo" referrerpolicy="no-referrer" src="../assets/bosch-header-logo.png"/>
    </div>

    <div class="headers">
      <div class="welcome">{{ $t('welcome') }}</div>
      <div class="login-title">{{ $t('appName') }}</div>
      <div class="points">······</div>
    </div>

    <div class="footer">
      <bosch-footer/>
    </div>
    <v-snackbar v-model="snackbar.visible" :timeout="2000" :color="snackbar.color" :top="true">
      {{ snackbar.text }}
    </v-snackbar>
    <login-dialog ref="loginPopup"/>
  </div>
</template>


<script>
import { loginByEmail } from '@/api/loginApi';
import BoschFooter from '@/components/BoschFooter.vue';
import { loadLanguageAsync } from '@/i18n';
import formRuleMixin from '@/mixins/formRuleMixin';
import constants from '@/utils/constants';
import helper from '@/utils/helper';
import { setLocalStorageStringByKey } from '@/utils/storage';
import LoginDialog from '@/views/loginDialog.vue';
import store from "@/store";

export default {
  mixins: [
    formRuleMixin,
  ],
  components: {
    BoschFooter,
    LoginDialog,
  },
  data() {
    return {
      emailValid: false,
      needBind: false,
      popupError: false,
      popupMsg: '',
      loginType: '',
      verifyImage: '',
      logining: false,
      snackbar: {
        visible: false,
        text: '',
        color: '',
      },
      loginForm: {
        userEmail: '',
        userPw: '',
        verifyCode: '',
        valid: true,
      },
      selectedLang: '',
    };
  },
  methods: {
    loadVerifyCodeImg() {
      this.emailValid = new RegExp(
        /^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/,
      ).test(this.loginForm.userEmail);
      if (this.emailValid) {
        this.verifyImage = `/api/verify-code?email=${this.loginForm.userEmail}&t=${new Date()}`;
      }
    },
    ssoLogin() {
      window.location.href = this.globalVars.CIAM_AUTH_URL;
    },
    adfsLogin() {
      window.location.href = this.globalVars.ADFS_AUTH_URL;
    },
    clearSsoLoginParams() {
      this.needBind = false;
      this.popupError = false;
      this.loginType = '';
    },
    async login() {
      const reg = new RegExp(
        /^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/,
      );
      // error and errorText not define, this code need remove
      if (this.loginForm.userEmail === '') {
        this.error = true;
        this.errorText = 'Email is required';
      } else if (!reg.test(this.loginForm.userEmail)) {
        this.error = true;
        this.errorText = 'Email must be valid';
      } else if (this.loginForm.userPw === '') {
        this.error = true;
        this.errorText = 'Password is required';
      } else {
        this.error = false;
        const data = await loginByEmail(
          {
            userEmail: this.loginForm.userEmail,
            userPw: this.loginForm.userPw,
            verifyCode: this.loginForm.verifyCode,
          },
        );
        if (data.code === 1200) {
          localStorage.setItem('x-token', data.data.token);
          this.$store.commit('set_token', data.data.token);
          this.$store.commit('updateUserName', data.data.user.name);
          this.loading = false;
          this.$router.push({ path: '/' });
        } else {
          this.snackbar.visible = true;
          this.snackbar.text = this.$t(helper.concatErrorString(data.message));
          this.snackbar.color = 'red';
          this.loadVerifyCodeImg();
        }
      }
    },
    loginWithAdfs(loginType) {
      this.globalVars.loginType = null;
      const needBind = $cookies.get('needBind');
      const error_code = $cookies.get('error_code');
      const error_msg = $cookies.get('error_msg');

      if (error_code && error_code !== '') {
        this.popupError = true;
        this.popupMsg = `${error_code}:${error_msg}`;
      } else if (needBind && needBind === 'true') {
        this.needBind = needBind;
      } else {
        const adfsToken = $cookies.get('adfs-token');
        const adfsTokenHint = $cookies.get('adfs-token-hint');
        localStorage.setItem('x-token', adfsToken);
        localStorage.setItem('adfsTokenHint', adfsTokenHint);
        localStorage.setItem('loginType', loginType);
        this.$store.commit('set_token', adfsToken);

        const updateUserName = $cookies.get('updateUserName');
        this.$store.commit('updateUserName', updateUserName);

        this.$router.push({ path: '/' });
      }
    },
    getLoginToken(code, state) {
      const loginParam = {
        body: {
          code,
          state,
        },
      };
      // 网络请求
      this.loading = true;
      this.globalVars.code = null;
      this.globalVars.state = null;
      const withCredentials = this.globalVars.CIAM_AUTH_CORS_BOOLEAN ? this.globalVars.CIAM_AUTH_CORS_BOOLEAN : false;
      this.$axios.post('/api/oauth2/login/get/token', loginParam,
        {
          withCredentials,
        })
        .then((res) => {
          if (res.data.status !== 1200) {
            this.logining = false;
            this.popupError = true;
            this.popupMsg = `${res.data.status}:${res.data.message}`;
            this.$refs.loginPopup.popupError = this.popupError;
            this.$refs.loginPopup.loginMsg = this.popupMsg;

            this.$refs.loginPopup.loginType = this.loginType;
            this.$refs.loginPopup.dialog = true;
          } else if (res.data.body.needBind && res.data.body.needBind === true) {
            this.logining = false;
            this.needBind = res.data.body.needBind;
            this.$refs.loginPopup.needBind = this.needBind;
            this.$refs.loginPopup.loginType = this.loginType;
            this.$refs.loginPopup.loginMsg = this.$t('signIn.notBindMobile');
            this.$refs.loginPopup.dialog = true;
            localStorage.setItem('tokenHint', res.data.body.tokenHint);
          } else {
            // 解析并且保存token
            const { token } = res.data.body;
            localStorage.setItem('tokenHint', res.data.body.tokenHint);
            localStorage.setItem('x-token', token);
            this.$store.commit('set_token', token);
            this.$store.commit('updateUserName', res.data.body.loginUser.name);
            this.loading = false;
            this.$router.push({ path: '/' });
          }
        });
    },
    async onUpdateLang(language) {
      setLocalStorageStringByKey(constants.CURRENT_LANGUAGE, language.value);
      await loadLanguageAsync(language.value);
      if (language.value === constants.EN_LANGUAGE) {
        this.selectedLang = this.$t('languageSwitch.english');
      } else if (language.value === constants.CN_LANGUAGE) {
        this.selectedLang = this.$t('languageSwitch.chinese');
      } else {
        this.selectedLang = this.$t('languageSwitch.malay');
      }
    },
  },
  mounted() {
    if (this.loginType && this.loginType === 'adfs') {
      if (this.popupError && this.popupError === 'true') {
        this.$refs.loginPopup.popupError = this.popupError;
        this.$refs.loginPopup.loginMsg = this.popupMsg;
        this.$refs.loginPopup.loginType = this.loginType;
        this.$refs.loginPopup.dialog = true;
      }
      if (this.needBind && this.needBind === 'true') {
        this.$refs.loginPopup.needBind = this.needBind;
        this.$refs.loginPopup.loginMsg = this.$t('signIn.notBindNT');
        this.$refs.loginPopup.loginType = this.loginType;
        this.$refs.loginPopup.dialog = true;
      }
    }
  },
  created() {
    const { code } = this.globalVars;
    const { state } = this.globalVars;
    const { loginType } = this.globalVars;
    store.commit('setClearMPTSearch', true);
    store.commit('setClearDTSearch', true);

    // check if token exist
    if (this.$store.state.token !== null && this.$store.state.token !== '') {
      this.loading = false;
      this.$router.push({ path: '/' });
    } else if (code && state && code !== '' && state !== '') {
      this.logining = true;
      this.loginType = 'ciam';
      this.getLoginToken(code, state);
    } else if (loginType && loginType === 'adfs') {
      this.loginType = loginType;
      this.loginWithAdfs(loginType);
    } else {
      this.logining = false;
    }

    if (this.$i18n.locale === constants.EN_LANGUAGE) {
      this.selectedLang = this.$t('languageSwitch.english');
    } else if (this.$i18n.locale === constants.CN_LANGUAGE) {
      this.selectedLang = this.$t('languageSwitch.chinese');
    } else {
      this.selectedLang = this.$t('languageSwitch.malay');
    }
  },
  computed: {
    languages() {
      return [
        {
          displayTxt: this.$t('languageSwitch.english'),
          value: constants.EN_LANGUAGE,
        },
        {
          displayTxt: this.$t('languageSwitch.chinese'),
          value: constants.CN_LANGUAGE,
        },
        {
          displayTxt: this.$t('languageSwitch.malay'),
          value: constants.MY_LANGUAGE,
        },
      ];
    },
  },
};
</script>

<style lang="scss" scoped>
</style>
