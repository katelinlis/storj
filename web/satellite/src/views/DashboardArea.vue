// Copyright (C) 2019 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div class="dashboard-container">
        <div v-if="isLoading" class="loading-overlay active">
            <img class="loading-image" src="@/../static/images/register/Loading.gif" alt="Company logo loading gif">
        </div>
        <div v-if="!isLoading" class="dashboard-container__wrap">
            <NavigationArea class="regular-navigation"/>
            <div class="dashboard-container__wrap__column">
                <DashboardHeader/>
                <div class="dashboard-container__main-area">
                    <div class="dashboard-container__main-area__bar-area">
                        <VInfoBar
                            v-if="isInfoBarShown"
                            :first-value="storageRemaining"
                            :second-value="bandwidthRemaining"
                            first-description="of Storage Remaining"
                            second-description="of Bandwidth Remaining"
                            :path="projectDetailsPath"
                            link="https://support.tardigrade.io/hc/en-us/requests/new?ticket_form_id=360000683212"
                            link-label="Request Limit Increase ->"
                        />
                    </div>
                    <div class="dashboard-container__main-area__content">
                        <router-view/>
                    </div>
                </div>
            </div>
            <div
                v-if="isBlurShown"
                class="dashboard-container__blur-area"
            >
                <div class="dashboard-container__blur-area__button" @click="onCreateButtonClick">
                    <span class="dashboard-container__blur-area__button__label">+ Create Project</span>
                </div>
                <div class="dashboard-container__blur-area__message-box">
                    <div class="dashboard-container__blur-area__message-box__left-area">
                        <AddImage/>
                        <span class="dashboard-container__blur-area__message-box__left-area__message">
                            Create your first project
                        </span>
                    </div>
                    <CloseCrossIcon class="dashboard-container__blur-area__message-box__close-cross-container" @click="onCloseClick"/>
                </div>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

import VInfoBar from '@/components/common/VInfoBar.vue';
import DashboardHeader from '@/components/header/HeaderArea.vue';
import NavigationArea from '@/components/navigation/NavigationArea.vue';

import CloseCrossIcon from '@/../static/images/common/closeCross.svg';
import AddImage from '@/../static/images/dashboard/Add.svg';

import { ErrorUnauthorized } from '@/api/errors/ErrorUnauthorized';
import { RouteConfig } from '@/router';
import { BUCKET_ACTIONS } from '@/store/modules/buckets';
import { PAYMENTS_ACTIONS } from '@/store/modules/payments';
import { PROJECTS_ACTIONS } from '@/store/modules/projects';
import { PROJECT_USAGE_ACTIONS } from '@/store/modules/usage';
import { USER_ACTIONS } from '@/store/modules/users';
import { CreditCard } from '@/types/payments';
import { Project } from '@/types/projects';
import { Size } from '@/utils/bytesSize';
import {
    API_KEYS_ACTIONS,
    APP_STATE_ACTIONS,
    PM_ACTIONS,
} from '@/utils/constants/actionNames';
import { AppState } from '@/utils/constants/appStateEnum';
import { ProjectOwning } from '@/utils/projectOwning';

const {
    SETUP_ACCOUNT,
    GET_BALANCE,
    GET_CREDIT_CARDS,
    GET_BILLING_HISTORY,
    GET_PROJECT_CHARGES_CURRENT_ROLLUP,
} = PAYMENTS_ACTIONS;

@Component({
    components: {
        NavigationArea,
        DashboardHeader,
        VInfoBar,
        AddImage,
        CloseCrossIcon,
    },
})
export default class DashboardArea extends Vue {
    /**
     * Holds router link to project details page.
     */
    public readonly projectDetailsPath: string = RouteConfig.ProjectDashboard.with(RouteConfig.ProjectDetails).path;

    /**
     * Lifecycle hook after initial render.
     * Pre fetches user`s and project information.
     */
    public async mounted(): Promise<void> {
        // TODO: combine all project related requests in one
        try {
            await this.$store.dispatch(USER_ACTIONS.GET);
        } catch (error) {
            if (!(error instanceof ErrorUnauthorized)) {
                await this.$store.dispatch(APP_STATE_ACTIONS.CHANGE_STATE, AppState.ERROR);
                await this.$notify.error(error.message);
            }

            setTimeout(async () => await this.$router.push(RouteConfig.Login.path), 1000);

            return;
        }

        let balance: number = 0;
        let creditCards: CreditCard[] = [];

        try {
            await this.$store.dispatch(SETUP_ACCOUNT);
            balance = await this.$store.dispatch(GET_BALANCE);
            creditCards = await this.$store.dispatch(GET_CREDIT_CARDS);
            await this.$store.dispatch(GET_BILLING_HISTORY);
            await this.$store.dispatch(GET_PROJECT_CHARGES_CURRENT_ROLLUP);
        } catch (error) {
            await this.$notify.error(error.message);
        }

        let projects: Project[] = [];

        try {
            projects = await this.$store.dispatch(PROJECTS_ACTIONS.FETCH);
        } catch (error) {
            await this.$notify.error(error.message);

            return;
        }

        if (!projects.length && !creditCards.length && balance === 0) {
            await this.$store.dispatch(APP_STATE_ACTIONS.CHANGE_STATE, AppState.LOADED_EMPTY);

            try {
                await this.$router.push(RouteConfig.Overview.path);
            } catch (error) {
                return;
            }

            return;
        }

        if (!projects.length) {
            await this.$store.dispatch(APP_STATE_ACTIONS.CHANGE_STATE, AppState.LOADED_EMPTY);

            if (this.$store.getters.isBonusCouponApplied) {
                await this.$store.dispatch(APP_STATE_ACTIONS.TOGGLE_CONTENT_BLUR);
            }

            if (!this.isRouteAccessibleWithoutProject()) {
                try {
                    await this.$router.push(RouteConfig.Account.with(RouteConfig.Billing).path);
                } catch (error) {
                    return;
                }
            }

            return;
        }

        await this.$store.dispatch(PROJECTS_ACTIONS.SELECT, projects[0].id);

        await this.$store.dispatch(PM_ACTIONS.SET_SEARCH_QUERY, '');
        try {
            await this.$store.dispatch(PM_ACTIONS.FETCH, 1);
        } catch (error) {
            await this.$notify.error(`Unable to fetch project members. ${error.message}`);
        }

        try {
            await this.$store.dispatch(PROJECTS_ACTIONS.GET_LIMITS, this.$store.getters.selectedProject.id);
        } catch (error) {
            await this.$notify.error(`Unable to fetch project limits. ${error.message}`);
        }

        try {
            await this.$store.dispatch(API_KEYS_ACTIONS.FETCH, 1);
        } catch (error) {
            await this.$notify.error(`Unable to fetch api keys. ${error.message}`);
        }

        try {
            await this.$store.dispatch(PROJECT_USAGE_ACTIONS.FETCH_CURRENT_ROLLUP);
        } catch (error) {
            await this.$notify.error(`Unable to fetch project usage. ${error.message}`);
        }

        try {
            await this.$store.dispatch(BUCKET_ACTIONS.FETCH, 1);
        } catch (error) {
            await this.$notify.error(`Unable to fetch buckets. ${error.message}`);
        }

        if (this.$store.getters.isBonusCouponApplied && !new ProjectOwning(this.$store).userHasOwnProject()) {
            await this.$store.dispatch(APP_STATE_ACTIONS.TOGGLE_CONTENT_BLUR);
        }

        await this.$store.dispatch(APP_STATE_ACTIONS.CHANGE_STATE, AppState.LOADED);
    }

    /**
     * Indicates if info bar is shown.
     */
    public get isInfoBarShown(): boolean {
        const isBillingPage = this.$route.name === RouteConfig.Billing.name;

        return isBillingPage && new ProjectOwning(this.$store).userHasOwnProject();
    }

    /**
     * Returns formatted string of remaining storage.
     */
    public get storageRemaining(): string {
        const storageUsed = this.$store.state.projectsModule.currentLimits.storageUsed;
        const storageLimit = this.$store.state.projectsModule.currentLimits.storageLimit;

        const difference = storageLimit - storageUsed;
        if (difference < 0) {
            return '0 Bytes';
        }

        const remaining = new Size(difference, 2);

        return `${remaining.formattedBytes}${remaining.label}`;
    }

    /**
     * Returns formatted string of remaining bandwidth.
     */
    public get bandwidthRemaining(): string {
        const bandwidthUsed = this.$store.state.projectsModule.currentLimits.bandwidthUsed;
        const bandwidthLimit = this.$store.state.projectsModule.currentLimits.bandwidthLimit;

        const difference = bandwidthLimit - bandwidthUsed;
        if (difference < 0) {
            return '0 Bytes';
        }

        const remaining = new Size(difference, 2);

        return `${remaining.formattedBytes}${remaining.label}`;
    }

    /**
     * Indicates if loading screen is active.
     */
    public get isLoading(): boolean {
        return this.$store.state.appStateModule.appState.fetchState === AppState.LOADING;
    }

    /**
     * Indicates if content blur shown.
     */
    public get isBlurShown(): boolean {
        return this.$store.state.appStateModule.appState.isContentBlurShown;
    }

    /**
     * Toggles create project popup showing.
     */
    public onCreateButtonClick(): void {
        this.onCloseClick();
        this.$store.dispatch(APP_STATE_ACTIONS.TOGGLE_NEW_PROJ);
    }

    /**
     * Hides blur.
     */
    public onCloseClick(): void {
        this.$store.dispatch(APP_STATE_ACTIONS.TOGGLE_CONTENT_BLUR);
    }

    /**
     * This method checks if current route is available when user has no created projects.
     */
    private isRouteAccessibleWithoutProject(): boolean {
        const availableRoutes = [
            RouteConfig.Account.with(RouteConfig.Billing).path,
            RouteConfig.Account.with(RouteConfig.Settings).path,
            RouteConfig.Overview.path,
        ];

        return availableRoutes.includes(this.$router.currentRoute.path.toLowerCase());
    }
}
</script>

<style scoped lang="scss">
    .dashboard-container {
        position: fixed;
        max-width: 100%;
        width: 100%;
        height: 100%;
        left: 0;
        top: 0;
        background-color: #f5f6fa;
        z-index: 10;

        &__wrap {
            display: flex;

            &__column {
                display: flex;
                flex-direction: column;
                width: 100%;
            }
        }

        &__main-area {
            position: relative;
            width: 100%;
            height: calc(100vh - 50px);
            overflow-y: scroll;
            display: flex;
            flex-direction: column;

            &__bar-area {
                flex: 0 1 auto;
            }

            &__content {
                flex: 1 1 auto;
            }
        }

        &__blur-area {
            position: fixed;
            max-width: 100%;
            width: 100%;
            height: 100%;
            left: 0;
            top: 0;
            background-color: rgba(12, 37, 70, 0.5);
            backdrop-filter: blur(4px);
            z-index: 20;

            &__button {
                position: fixed;
                top: 30px;
                right: 148px;
                display: flex;
                align-items: center;
                justify-content: center;
                width: 156px;
                height: 40px;
                background-color: #fff;
                border: 1px solid #2683ff;
                border-radius: 6px;
                cursor: pointer;
                z-index: 21;

                &__label {
                    font-family: 'font_medium', sans-serif;
                    font-size: 15px;
                    line-height: 22px;
                    color: #2683ff;
                }
            }

            &__message-box {
                background-image: url('../../static/images/dashboard/message.png');
                background-size: 100% 100%;
                height: auto;
                width: auto;
                position: fixed;
                top: 80px;
                right: 100px;
                display: flex;
                align-items: center;
                justify-content: space-between;
                z-index: 21;
                padding: 30px 30px 20px 20px;

                &__left-area {
                    display: flex;
                    align-items: center;
                    justify-content: space-between;

                    &__message {
                        font-family: 'font_regular', sans-serif;
                        font-size: 14px;
                        line-height: 19px;
                        color: #373737;
                        margin-left: 15px;
                    }
                }

                &__close-cross-container {
                    display: flex;
                    justify-content: center;
                    align-items: center;
                    height: 17px;
                    width: 17px;
                    cursor: pointer;
                    margin-left: 50px;

                    &:hover .close-cross-svg-path {
                        fill: #2683ff;
                    }
                }
            }
        }
    }

    @media screen and (max-width: 1280px) {

        .regular-navigation {
            display: none;
        }

        .dashboard-container {

            &__blur-area {

                &__button {
                    right: 123px;
                }

                &__message-box {
                    right: 76px;
                }
            }
        }
    }

    @media screen and (max-width: 720px) {

        .dashboard-container {

            &__main-area {
                left: 60px;
            }
        }
    }

    .loading-overlay {
        display: flex;
        justify-content: center;
        align-items: center;
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        height: 100vh;
        z-index: 100;
        background-color: rgba(134, 134, 148, 1);
        visibility: hidden;
        opacity: 0;
        -webkit-transition: all 0.5s linear;
        -moz-transition: all 0.5s linear;
        -o-transition: all 0.5s linear;
        transition: all 0.5s linear;

        .loading-image {
            z-index: 200;
        }
    }

    .loading-overlay.active {
        visibility: visible;
        opacity: 1;
    }
</style>
