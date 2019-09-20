<template>
  <div class="verify-and-submit">
    <div class="row margin--top--lg margin--bottom">
      <div class="col">
        <h6>Submit for Review</h6>

        <div v-if="stage === stages.verify">
          <form>
            <label for="contributor-email"
              >Please enter your government email for verification</label
            >
            <div class="field has-addons">
              <div class="control expanded">
                <input
                  id="contributor-email"
                  name="email"
                  class="input"
                  :class="{ 'is-danger': errors.email }"
                  type="email"
                  placeholder="me@.gov.sg"
                  v-model="email"
                  @blur="validateEmail"
                  required
                />
              </div>
              <div class="control">
                <button
                  type="submit"
                  class="sgds-button is-primary"
                  @click.prevent="requestOtp"
                  :disabled="!emailRegex.test(email)"
                >
                  Send OTP
                </button>
              </div>
            </div>
            <p class="help is-danger" v-if="errors.email">{{ errors.email }}</p>
          </form>
        </div>

        <div v-if="stage === stages.submit">
          <form>
            <label for="otp"
              >Please enter the OTP sent to your email at {{ email }}</label
            >
            <div class="field has-addons">
              <div class="control expanded">
                <input
                  type="number"
                  name="otp"
                  id="otp"
                  class="input"
                  :class="{ 'is-danger': errors.otp }"
                  v-model="otp"
                  required
                />
              </div>
              <div class="control is-flex">
                <button
                  type="button"
                  class="sgds-button"
                  @click.prevent="stage = stages.verify"
                >
                  <i class="fas fa-undo"></i>
                </button>
                <button
                  type="submit"
                  class="sgds-button is-primary"
                  :disabled="!this.otp || this.otp.length !== 6"
                  @click.prevent="submit"
                >
                  Submit Changes
                </button>
              </div>
            </div>
            <p class="help is-danger" v-if="errors.otp">{{ errors.otp }}</p>
          </form>
        </div>

        <small>
          You will be able to track submission status at our GitHub repository's
          <a href="https://github.com/govtechsg/developer.gov.sg/pulls"
            >pull requests</a
          >
        </small>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import Noty from "noty";
import { emailRegex } from "./index";

const stages = {
  verify: "verify",
  submit: "submit"
};

export default {
  data() {
    return {
      emailRegex,
      stages,
      stage: stages.verify,
      email: null,
      otp: null,
      errors: {
        email: null,
        otp: null
      }
    };
  },
  methods: {
    validateEmail() {
      if (!this.email || !emailRegex.test(this.email)) {
        this.errors.email = "Please enter a valid government email";
        return false;
      }
      this.errors.email = null;
      return true;
    },
    requestOtp() {
      this.$emit("validate"); // Inform outer form to perform validation
      if (!this.validateEmail()) {
        return;
      }
      this.$emit("loading", true);
      axios
        .post("/.netlify/functions/api/request-otp", {
          email: this.email
        })
        .then(response => {
          new Noty({
            type: "success",
            text: `An OTP has been sent to ${this.email}. Please enter it before submitting your edits.`
          }).show();
          let otpRequestId = response.data.id;
          localStorage.setItem("otpRequestId", otpRequestId);
          this.stage = stages.submit;
        })
        .catch(error => {
          new Noty({
            type: "error",
            text: `An error has occurred: ${error.message || error}`
          }).show();
        })
        .finally(() => {
          this.$emit("loading", false);
        });
    },
    submit() {
      if (!this.otp || this.otp.length !== 6) {
        this.errors.otp = "Please a valid 6-digit OTP.";
        return;
      }
      let otpRequestId = localStorage.getItem("otpRequestId");
      if (!otpRequestId) {
        new Noty({
          type: "error",
          text: `Could not send submission; have you requested for your OTP?`
        }).show();
        return;
      }
      this.$emit("submit", {
        email: this.email,
        otp: this.otp,
        otpRequestId
      });
      this.otp = null;
    }
  }
};
</script>

<style scoped>
.expanded {
  flex: 1 1 auto;
}
</style>