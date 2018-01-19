# roguelikearcher

package es.albakin.roguelikearcher.screens;

import com.badlogic.gdx.Gdx;
import com.badlogic.gdx.Input.Keys;
import com.badlogic.gdx.assets.AssetManager;
import com.badlogic.gdx.audio.Music;
import com.badlogic.gdx.graphics.Color;
import com.badlogic.gdx.graphics.Cursor;
import com.badlogic.gdx.graphics.Pixmap;
import com.badlogic.gdx.graphics.Texture;
import com.badlogic.gdx.graphics.g2d.SpriteBatch;
import com.badlogic.gdx.graphics.g2d.TextureAtlas;
import com.badlogic.gdx.graphics.glutils.ShapeRenderer;
import com.badlogic.gdx.graphics.glutils.ShapeRenderer.ShapeType;
import com.badlogic.gdx.maps.tiled.TmxMapLoader;
import com.badlogic.gdx.maps.tiled.renderers.OrthogonalTiledMapRenderer;
import com.badlogic.gdx.math.Vector2;
import com.badlogic.gdx.math.Vector3;
import com.badlogic.gdx.physics.box2d.Body;
import com.badlogic.gdx.physics.box2d.Box2DDebugRenderer;
import com.badlogic.gdx.physics.box2d.World;
import com.badlogic.gdx.scenes.scene2d.Group;
import com.badlogic.gdx.utils.Array;

import org.json.JSONException;
import org.json.JSONObject;

import java.util.HashMap;

import es.albakin.roguelikearcher.MainGame;
import es.albakin.roguelikearcher.archer.Archer;
import es.albakin.roguelikearcher.archer.EnemyArcher;
import es.albakin.roguelikearcher.items.Arrow;
import es.albakin.roguelikearcher.utils.CirclePoint;
import es.albakin.roguelikearcher.utils.HUD;
import es.albakin.roguelikearcher.utils.MiniMap;
import es.albakin.roguelikearcher.utils.RadiusTimer;
import es.albakin.roguelikearcher.utils.ScreenTemplate;
import es.albakin.roguelikearcher.utils.StageProcessor;
import es.albakin.roguelikearcher.utils.WorldContactListener;
import es.albakin.roguelikearcher.utils.WorldGenerator;
import io.socket.client.IO;
import io.socket.client.Socket;
import io.socket.emitter.Emitter;
import jdk.nashorn.api.scripting.JSObject;

public class Screen1 extends ScreenTemplate {
	private final float UPDATE_TIME = 1/30f;
	float timer;
	Archer archer;
	Arrow arrow;
	Vector3 mouse_position;
	Vector3 mouse_position2;
	Cursor customCursor;
	RadiusTimer timerRadius;
	Pixmap pm;
	Group archers;
	Group missiles;
	Group weapons;
	Group enemies;
	SpriteBatch miniBatch;
	Vector2 startPosition;
	Array<Body> destroyBodies;
	Array<Group> groups;
	CirclePoint circlePoint;

	HashMap<String,EnemyArcher> enemyPlayers;



	
	MiniMap miniMap;
	int[] textureLayers = { 0 };
	int[] behindLayers = { 1 };
	int[] frontLayers = { 2 };
	boolean debugging = false;
	boolean showMinimap = false;
	float mapLimitRadius=55f;
	private WorldContactListener contact;
	private ShapeRenderer shapeRenderer;
	private HUD hud;
	Socket socket;
	private boolean inside;
	private Music music;
	private long init;
	private long now;
	private long diference;
	boolean threadON;

	public Screen1(MainGame game) {
		super(game);


		threadON = true;
		pm = new Pixmap(Gdx.files.internal("cross.png"));
		customCursor = Gdx.graphics.newCursor(pm, pm.getWidth() / 2, pm.getHeight() / 2);
		prefs = Gdx.app.getPreferences("GamePreferences");
		mouse_position = new Vector3(0, 0, 0);
		manager = new AssetManager();
		manager.load("arrow.png", Texture.class);
		manager.load("paraflechas.png", Texture.class);
		manager.load("archer.pack", TextureAtlas.class);
		manager.load("archer2.pack", TextureAtlas.class);
		manager.load("bow.pack", TextureAtlas.class);
		manager.load("shadow.png",Texture.class);
		manager.load("tiles/world.png",Texture.class);
		//manager.load("Music/music.mp3", Music.class);
		manager.finishLoading();
		
		
		destroyBodies = new Array<Body>();
		world = new World(new Vector2(0, 0), true);
		b2dr = new Box2DDebugRenderer();
		stage = new StageProcessor(viewport, batch,this);
		hud = new HUD(manager);
		Gdx.input.setInputProcessor(stage);
		//music = manager.get("Music/music.mp3", Music.class);
		//music.setLooping(true);

		archers = new Group();
		enemies = new Group();
		missiles = new Group();
		weapons = new Group();
		enemyPlayers = new HashMap<String,EnemyArcher>();
		groups = new Array<Group>();
		groups.add(archers);
		groups.add(enemies);
		groups.add(missiles);
		groups.add(weapons);
		
		
		
		contact = new WorldContactListener(archer, manager, this);
		maploader = new TmxMapLoader();
		map = maploader.load("tiles/world.tmx");
		
		
		miniMap = new MiniMap(manager);
		shapeRenderer = new ShapeRenderer();
		renderer = new OrthogonalTiledMapRenderer(map, 1 / MainGame.PPM);
		generator = new WorldGenerator(this);
		startPosition = generator.getStartPosition();
		archer = new Archer(world, manager,archers, cam, startPosition.x, startPosition.y,weapons,missiles,manager);
		
		cam.position.set(archer.body.getPosition().x ,archer.body.getPosition().y, 0);
		cam.update();
		world.setContactListener(contact);
		/*new Paraflechas(world, manager, cam, enemies,missiles, startPosition.x -50, startPosition.y, archer);
		new Paraflechas(world, manager, cam, enemies,missiles, startPosition.x+50, startPosition.y ,archer);*/
		
		stage.addActor(enemies);
		stage.addActor(weapons);
		stage.addActor(missiles);
		stage.addActor(archers);
		
		timerRadius = new RadiusTimer(mapLimitRadius,20);

		init = System.currentTimeMillis();
		connectSocket();
		configSocketEvents();

		
		
		
		
	}


	public void updateServer(float dt){
		timer+=dt;
		if (timer >= UPDATE_TIME && archer!= null&& archer.hasMoved() ){
			timer = 0;
			JSONObject data = new JSONObject();
			try{

				data.put("x",archer.sprite.getX());
				data.put("y",archer.sprite.getY());
				data.put("rot",archer.bow.rot);
			}catch(JSONException e){
				System.out.print("Error Sending JSON");
			}
			socket.emit("actualiza",data);
		}

	}




	private void connectSocket() {
		System.out.print("Conectando socket");
		try{
			socket = IO.socket("http://127.0.0.1:9095");
			socket.connect();


		}catch (Exception e){
			System.out.print("Connection Error");
		}
	}
	private void configSocketEvents() {
		socket.on(Socket.EVENT_CONNECT, new Emitter.Listener() {
			@Override
			public void call(Object... args) {


			}
		}).on(Socket.EVENT_DISCONNECT, new Emitter.Listener() {
			@Override
			public void call(Object... args) {

			}
		}).on("socketID", new Emitter.Listener() {
			@Override
			public void call(Object... args) {
				JSONObject data = (JSONObject) args[0];
				try {
					String id = data.getString("id");
					System.out.println("ID: " + id);
				} catch (JSONException e) {
					e.printStackTrace();
				}
			}
		}).on("newPlayer", new Emitter.Listener() {
			@Override
			public void call(Object... args) {
				JSONObject data = (JSONObject) args[0];
				try {
					String id = data.getString("id");
					System.out.println("New Player Connected ID: " + id);
					enemyPlayers.put(id, new EnemyArcher(world, manager, enemies, cam, startPosition.x, startPosition.y, weapons, missiles, manager));
				} catch (JSONException e) {
					e.printStackTrace();
				}
			}
		}).on("actualiza", new Emitter.Listener() {
			@Override
			public void call(Object... args) {
				JSONObject data = (JSONObject) args[0];
				try {
					String playerId = data.getString("id");
					Double x = data.getDouble("x");
					Double y = data.getDouble("y");
					Double rot = data.getDouble("rot");
					if (enemyPlayers.get(playerId) != null) {
						enemyPlayers.get(playerId).sprite.setPosition(x.floatValue(), y.floatValue());
						enemyPlayers.get(playerId).bow.rot = rot.floatValue();
					}

				} catch (JSONException e) {
					e.printStackTrace();
				}
			}
		});}
		@Override
		public void update (float delta){
			updateServer(delta);
			timerRadius.update();
			hud.seconds = timerRadius.timetoDecrese;
			hud.updateSeconds(isOutRadius(), true);
			mapLimitRadius = timerRadius.number;

			Gdx.graphics.setCursor(customCursor);
			stage.act(delta);
			hud.stage.act(delta);
			renderer.setView(cam);
			this.mouse_position = new Vector3(Gdx.input.getX(), Gdx.input.getY(), 0);
			Vector2 normalizedScreenCoords = new Vector2(mouse_position.x / Gdx.graphics.getWidth(), mouse_position.y / Gdx.graphics.getHeight());
			normalizedScreenCoords.sub(.5f, .5f);
			float offset = 0.8f;

			cam.position.set(archer.body.getPosition().x + normalizedScreenCoords.x * offset, archer.body.getPosition().y - normalizedScreenCoords.y * offset, 0);
			cam.update();
			miniMap.update();


			world.step(1 / 60f, 6, 2);
			inputs();
			world.getBodies(destroyBodies);
			for (Body body : destroyBodies) {
				if (body.getUserData() == "delete") {
					world.destroyBody(body);
				}
			}

		}






	private void inputs() {
		if (Gdx.input.isKeyPressed(Keys.A)) {
			Vector2 vel;
			vel = archer.body.getLinearVelocity();
			vel.x = -archer.speed;
			archer.body.setLinearVelocity(vel);

		}
		if (Gdx.input.isKeyPressed(Keys.D)) {
			Vector2 vel;
			vel = archer.body.getLinearVelocity();
			vel.x = archer.speed;
			archer.body.setLinearVelocity(vel);

		}
		if (Gdx.input.isKeyPressed(Keys.S)) {
			Vector2 vel;
			vel = archer.body.getLinearVelocity();
			vel.y = -archer.speed;
			archer.body.setLinearVelocity(vel);

		}
		if (Gdx.input.isKeyPressed(Keys.W)) {
			Vector2 vel;
			vel = archer.body.getLinearVelocity();
			vel.y = archer.speed;
			archer.body.setLinearVelocity(vel);

		}
		if (Gdx.input.isKeyJustPressed(Keys.X)) {
			
			if(!debugging){
				for(Group group: groups){
					group.setDebug(true,true);
				}
				debugging = true;
			}else{
				for(Group group: groups){
					group.setDebug(false,true);
				}
				debugging = false;
			}
			

		}
		if (Gdx.input.isKeyJustPressed(Keys.Q)) {
			if (showMinimap){
				showMinimap = false;
			}else{
				showMinimap = true;
			}
		}

		if (Gdx.input.isKeyJustPressed(Keys.ESCAPE)) {
			Gdx.app.exit();
		}

		if (!Gdx.input.isKeyPressed(Keys.W) && !Gdx.input.isKeyPressed(Keys.S)) {
			Vector2 vel;
			vel = archer.body.getLinearVelocity();
			vel.y = 0;
			archer.body.setLinearVelocity(vel);
		}
		if (!Gdx.input.isKeyPressed(Keys.A) && !Gdx.input.isKeyPressed(Keys.D)) {
			Vector2 vel;
			vel = archer.body.getLinearVelocity();
			vel.x = 0;
			archer.body.setLinearVelocity(vel);
		}
		

	}

	public void Atack(){
		this.mouse_position2 = new Vector3(Gdx.input.getX(), Gdx.input.getY(), 0);
		cam.unproject(mouse_position2);
		archer.atack(mouse_position2);
	}

	@Override
	public void draw() {
		
		renderer.render(this.textureLayers);
		renderer.render(this.behindLayers);
		
		stage.draw();
		
		renderer.render(this.frontLayers);
		Gdx.gl.glLineWidth(20);
		 shapeRenderer.setProjectionMatrix(cam.combined);
		 shapeRenderer.begin(ShapeType.Line);
		 shapeRenderer.setColor(Color.BLUE);
		 shapeRenderer.circle(40,40,mapLimitRadius,1000);
		 shapeRenderer.end();
		 hud.stage.draw();
		
		if (debugging)
		b2dr.render(world, cam.combined);
		if(showMinimap){
			miniMap.render();
			
			 shapeRenderer.setProjectionMatrix(miniMap.miniCam.combined);
			 shapeRenderer.begin(ShapeType.Filled);
			 shapeRenderer.setColor(Color.RED);
			 shapeRenderer.circle(archer.body.getPosition().x,archer.body.getPosition().y,0.5f,500);
			 shapeRenderer.setColor(Color.BLUE);
			 shapeRenderer.circle(40,40,0.6f,500);
			 shapeRenderer.end();
			 shapeRenderer.begin(ShapeType.Line);
			 shapeRenderer.setColor(Color.BLUE);
			 shapeRenderer.circle(40,40,mapLimitRadius,500);
			 shapeRenderer.end();
		}
	
	}
	public boolean isOutRadius(){
		
		if (Math.sqrt (Math.pow((40-archer.body.getPosition().x),2) + Math.pow((40-archer.body.getPosition().y), 2)) >= (mapLimitRadius)){
			return true;
			
		}else{
			return false;
		}
		
	}

	@Override
	public void restartScreen() {
		// TODO Auto-generated method stub

	}
	 public void resize(int width, int height) {
		 stage.getViewport().update(width, height);
	   }
	@Override
	public void goMenu() {
		// TODO Auto-generated method stub

	}

	@Override
	public void levelEnd() {
		// TODO Auto-generated method stub

	}

	@Override
	public void dispose() {
		this.customCursor.dispose();
		stage.dispose();
		map.dispose();
		pm.dispose();

	}

}



//PROBLEMATIC CODE
public void updateServer(float dt){
		timer+=dt;
		if (timer >= UPDATE_TIME && archer!= null&& archer.hasMoved() ){
			timer = 0;
			JSONObject data = new JSONObject();
			try{

				data.put("x",archer.sprite.getX());
				data.put("y",archer.sprite.getY());
				data.put("rot",archer.bow.rot);
			}catch(JSONException e){
				System.out.print("Error Sending JSON");
			}
			socket.emit("actualiza",data);
		}

	}
//Lag 2ms per each 
