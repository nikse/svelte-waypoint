<script context="module">
  let innerHeight;
</script>

<script>
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  export let offset = 0;
  export let throttle = 250;
  export let style = '';
  export let once = true;
  export let threshold = 0;
  export let disabled = false;

  let className = '';
  export { className as class };

  let visible = disabled;
  let wasVisible = false;
  let intersecting = false;
  let removeHandlers = () => {};

  function throttleFn(fn, time) {
    let last, deferTimer;

    return () => {
      const now = +new Date();

      if (last && now < last + time) {
        // hold on to it
        clearTimeout(deferTimer);
        deferTimer = setTimeout(function () {
          last = now;
          fn();
        }, time);
      } else {
        last = now;
        fn();
      }
    };
  }

  function callEvents(wasVisible, observer, node) {
    if (visible && !wasVisible) {
      dispatch('enter');
      return;
    }

    if (wasVisible && !intersecting) {
      dispatch('leave');
    }

    if (once && wasVisible && !intersecting) {
      removeHandlers();
    }
  }

  function calculateOffset() {
    if (typeof innerHeight === 'undefined') {
      innerHeight = window.innerHeight || document.documentElement.clientHeight;
    }

    return /vh/.test(offset)
      ? (parseInt(offset.match(/-?\d*/)[0], 10) / 100) * innerHeight
      : offset;
  }

  function waypoint(node) {
    if (!window || disabled) return;

    if (window.IntersectionObserver && window.IntersectionObserverEntry) {
      const observer = new IntersectionObserver(
        ([{ isIntersecting }]) => {
          wasVisible = visible;
          intersecting = isIntersecting;

          if (wasVisible && once && !isIntersecting) {
            callEvents(wasVisible, observer, node);
            return;
          }

          visible = isIntersecting;

          callEvents(wasVisible, observer, node);
        },
        {
          rootMargin: `${calculateOffset()}px`,
          threshold,
        }
      );

      observer.observe(node);

      removeHandlers = () => observer.unobserve(node);

      return {
        destroy() {
          removeHandlers();
        },
      };
    }

    function checkIsVisible() {
      // Kudos https://github.com/twobin/react-lazyload/blob/master/src/index.jsx#L93
      if (
        !(node.offsetWidth || node.offsetHeight || node.getClientRects().length)
      )
        return;

      let top;
      let height;
      const visibleOffset = calculateOffset();

      try {
        ({ top, height } = node.getBoundingClientRect());
      } catch (e) {
        ({ top, height } = defaultBoundingClientRect);
      }

      wasVisible = visible;
      intersecting =
        top - visibleOffset <= innerHeight && top + height + visibleOffset >= 0;

      if (wasVisible && once && !isIntersecting) {
        callEvents(wasVisible, observer, node);
        return;
      }

      visible = intersecting;

      callEvents(wasVisible);
    }

    checkIsVisible();

    const throttled = throttleFn(checkIsVisible, throttle);

    window.addEventListener('scroll', throttled);
    window.addEventListener('resize', throttled);

    removeHandlers = () => {
      window.removeEventListener('scroll', throttled);
      window.removeEventListener('resize', throttled);
    };

    return {
      destroy() {
        removeHandlers();
      },
    };
  }
</script>

<svelte:window bind:innerHeight />

<div class={className} {style} use:waypoint>
  <slot />
</div>
