<div class="skeleton">
  <div class="children">
  </div>
</div>

<style>
  .skeleton {
    height: 100%;
    width:100%;
    align-self: center;
    opacity: 0;
    display: flex;
    align-items: stretch;
    justify-content: stretch;
    background-color: var(--spectrum-global-color-gray-200) !important;
    overflow: hidden;
    position: relative;
    animation: fadeIn 130ms ease 0s 1 normal forwards;
  }
  .children {
    flex: auto;
    pointer-events: none;
    opacity: 0.65;
    background-image: linear-gradient(
      90deg,
      rgba(255, 255, 255, 0) 0,
      rgba(255, 255, 255, 0.15) 20%,
      rgba(255, 255, 255, 0.3) 60%,
      rgba(255, 255, 255, 0)
    );
    animation: shimmer 2s infinite;
  }

  @keyframes fadeIn {
    0% {
      opacity: 0;
    }
    100% {
      opacity: 0.55;
    }
  }
  @keyframes shimmer {
    100% {
      transform: translateX(100%);
    }
  }
</style>