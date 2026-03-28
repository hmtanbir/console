<script lang="ts">
    import { goto, invalidate } from '$app/navigation';
    import { base } from '$app/paths';
    import {
        Button,
        Form,
        InputChoice,
        InputEmail,
        InputPassword,
        InputText
    } from '$lib/elements/forms';
    import { addNotification } from '$lib/stores/notifications';
    import { sdk } from '$lib/stores/sdk';
    import { Unauthenticated } from '$lib/layout';
    import { Dependencies } from '$lib/constants';
    import { Submit, trackError, trackEvent } from '$lib/actions/analytics';
    import { ID, OAuthProvider } from '@appwrite.io/console';
    import { isCloud } from '$lib/system';
    import { page } from '$app/state';
    import { redirectTo } from '$routes/store';
    import { checkPricingRefAndRedirect } from '$lib/helpers/pricingRedirect';
    import { Layout, Link, Typography } from '@appwrite.io/pink-svelte';
    import { getRandomTestimonial } from '$lib/data/testimonials';

    export let data;

    let name: string, mail: string, pass: string, disabled: boolean;
    let terms = false;

    const randomTestimonial = getRandomTestimonial();
    const testimonialCampaign = {
        $id: 'testimonial-signup',
        template: 'review',
        title: randomTestimonial.headline,
        description: 'Join thousands of developers building amazing apps with Appwrite',
        reviews: [
            {
                name: randomTestimonial.name,
                image: randomTestimonial.avatar,
                description: randomTestimonial.title,
                review: randomTestimonial.blurb
            }
        ]
    };

    trackEvent(Submit.TestimonialView, {
        testimonial_id: randomTestimonial.id,
        testimonial_name: randomTestimonial.name,
        testimonial_company: randomTestimonial.title
    });

    async function register() {
        try {
            disabled = true;
            await sdk.forConsole.account.create({
                userId: ID.unique(),
                email: mail,
                password: pass,
                name: name ?? ''
            });
            await sdk.forConsole.account.createEmailPasswordSession({
                email: mail,
                password: pass
            });

            trackEvent(Submit.AccountCreate, {
                campaign_name: data?.couponData?.code,
                email: mail,
                name: name,
                testimonial_id: randomTestimonial.id,
                testimonial_name: randomTestimonial.name
            });

            if (data?.couponData?.code) {
                await goto(`${base}/apply-credit?code=${data?.couponData?.code}`);
                return;
            } else if (data?.campaign?.$id) {
                await goto(`${base}/apply-credit?campaign=${data.campaign.$id}`);
                return;
            } else if ($redirectTo) {
                window.location.href = $redirectTo;
                return;
            } else if (page.url.searchParams) {
                const redirect = page.url.searchParams.get('redirect');
                page.url.searchParams.delete('redirect');
                if (redirect) {
                    await goto(`${redirect}${page.url.search}`);
                } else if (isCloud) {
                    checkPricingRefAndRedirect(page.url.searchParams);
                } else {
                    await goto(`${base}/${page.url.search ?? ''}`);
                }
            } else {
                await goto(base);
            }

            await invalidate(Dependencies.ACCOUNT);
        } catch (error) {
            disabled = false;
            addNotification({
                type: 'error',
                message: error.message
            });
            trackError(error, Submit.AccountCreate);
        }
    }

    function onGithubLogin() {
        let successUrl = window.location.origin;

        if (page.url.searchParams.has('code')) {
            successUrl += `?code=${page.url.searchParams.get('code')}`;
        } else if (page.url.searchParams.has('campaign')) {
            successUrl += `?campaign=${page.url.searchParams.get('campaign')}`;
        }

        sdk.forConsole.account.createOAuth2Session({
            provider: OAuthProvider.Github,
            success: successUrl,
            failure: window.location.origin,
            scopes: ['read:user', 'user:email']
        });
    }
</script>

<svelte:head>
    <title>Sign up - Appwrite</title>
</svelte:head>

<Unauthenticated coupon={data?.couponData} campaign={data?.campaign || testimonialCampaign}>
    <svelte:fragment slot="title">Appwrite</svelte:fragment>
    <svelte:fragment slot="links">
        <Layout.Stack direction="row" justifyContent="center" alignItems="center">
            Already have an account? <Link.Anchor href={`${base}/login`} variant="quiet">Sign in</Link.Anchor>
        </Layout.Stack>
    </svelte:fragment>
</Unauthenticated>
