<script lang="ts">
  import Eventhub from 'eventhub-jsclient';
  import { onMount } from 'svelte';

  const ev = new Eventhub('ws://localhost:8080');
  let cursor = { x: 0, y: 0 };
  let cursors = new Map<
    string,
    { x: number; y: number; targetX: number; targetY: number }
  >();
  let cursorArray = [];

  function createRandomString(length: number) {
    const chars =
      'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    let result = '';
    for (let i = 0; i < length; i++) {
      result += chars.charAt(Math.floor(Math.random() * chars.length));
    }
    return result;
  }

  const userID = createRandomString(10);

  onMount(() => {
    window.addEventListener('mousemove', updateCursorPosition);

    ev.connect()
      .then(() => {
        console.log('Connected');
        ev.subscribe('cursors', function (msg) {
          const { id, x, y } = JSON.parse(msg.message);
          if (!cursors.has(id)) {
            cursors.set(id, { x, y, targetX: x, targetY: y });
          } else {
            const cursor = cursors.get(id);
            cursor.targetX = x;
            cursor.targetY = y;
          }
          cursorArray = Array.from(cursors.entries());
          console.log(id);
        });

        // Start the LERP loop
        requestAnimationFrame(lerpCursors);
      })
      .catch((err) => {
        console.log(`Error connecting to Eventhub: ${err}`);
      });
  });

  const updateCursorPosition = (event: MouseEvent) => {
    cursor.x = event.clientX;
    cursor.y = event.clientY;
    ev.publish(
      'cursors',
      JSON.stringify({
        id: `user-${userID}`,
        x: cursor.x,
        y: cursor.y,
      })
    );
  };

  function lerpCursors() {
    const lerpFactor = 0.1; // Adjust for smoothing speed; lower is slower

    for (const [id, cursor] of cursors) {
      cursor.x += (cursor.targetX - cursor.x) * lerpFactor;
      cursor.y += (cursor.targetY - cursor.y) * lerpFactor;
    }

    cursorArray = Array.from(cursors.entries()); // Update cursorArray for reactivity
    requestAnimationFrame(lerpCursors); // Keep animating
  }
</script>

<main class="bg-gray-900 w-screen h-screen relative">
  {#each cursorArray as [id, position]}
    <div
      class="absolute flex -translate-x-5 -translate-y-5 space-x-2 items-end"
      style="left: {position.x}px; top: {position.y}px;"
    >
      <div class="w-10 h-10 bg-red-500 rounded-full"></div>
      <p class="text-gray-100">{id}</p>
    </div>
  {/each}
</main>
