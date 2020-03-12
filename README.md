# howToUseFirebase_Storage&DB-on-Android
### ※파일 업로드  
<a href=https://firebase.google.com/docs/storage/android/upload-files?hl=ko>참조 문서</a>
우선 버킷주소를 참조할 객체를 선언한다.  
```
FirebaseStorage storage;
storage=FirebaseStorage.getInstance();
```  
  
그 후, 디바이스의 갤러리에서 사진을 pick할 수 있는곳으로 이동한다.  
★ <b>코드를 지켜주기 바람.</b>  
```
Intent intent=new Intent();
intent.setType(MediaStore.Images.Media.CONTENT_TYPE);
intent.setAction(Intent.ACTION_PICK);
startActivityForResult(intent,101);
```  
  
가져온 사진의 경로를 "content://" 타입이 아닌 다른 경로타입으로 바꾸어 주고 업로드 한다.  
★ <b>매우 중요한 부분이다. 기존 경로는 사용할 수 가 없으므로 꼭 변환필요 </b> 
```
@Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
       super.onActivityResult(requestCode, resultCode, data);
       if(requestCode==101 && resultCode==RESULT_OK){
           String path=getPath(data.getData());
           StorageReference reference=storage.getReferenceFromUrl("bucket address"); //스토리지 버킷주소 참조

           Uri file=Uri.fromFile(new File(path)); //해당 파일의 경로
           StorageReference putref=reference.child("images/"+file.getLastPathSegment()); //기존 reference의 참조주소에서 해당 하위트리 참조
           UploadTask task=putref.putFile(file); //해당 경로에 파일을 삽입

           task.addOnFailureListener(new OnFailureListener() { //실패시 이벤트
               @Override
               public void onFailure(@NonNull Exception e) {
                   //DoSomething if fail
               }
           }).addOnSuccessListener(new OnSuccessListener<UploadTask.TaskSnapshot>() { //성공시 이벤트
               @Override
               public void onSuccess(UploadTask.TaskSnapshot taskSnapshot) {
                   //taskSnapshot.getMetadata() 는 파일의 메타데이터를 포함함(사이즈 등등)
                   //DoSomething if success
                   Toast.makeText(HomeActivity.this,"파일 업로드 성공",Toast.LENGTH_SHORT).show();
               }
           });
       }
    }
    public String getPath(Uri uri){ //적절한 path로 바꿔주는 코드
        String[] proj={MediaStore.Images.Media.DATA};
        CursorLoader cursorLoader=new CursorLoader(this,uri,proj,null,null,null);
        Cursor cursor=cursorLoader.loadInBackground();
        int index=cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATA);

        cursor.moveToFirst();

        return cursor.getString(index);
    }
```  
  
### ※Do you want to use Database?(Module:app gradle 12.0.1)  
만일 이미지를 업로드와 동시에 이미지의 정보를 데이터베이스에 저장을 원할시, 위 코드에서
```
        Uri file=Uri.fromFile(new File(path)); //해당 파일의 경로
        
        //기존 reference의 참조주소에서 해당 하위트리를 침조
        final StorageReference putref=reference.child("images/"+file.getLastPathSegment());
        UploadTask task=putref.putFile(file); //해당 경로에 파일을 삽입

        task.addOnFailureListener(new OnFailureListener() {
            @Override
            public void onFailure(@NonNull Exception e) {
                //DoSomething if fail
            }
        }).addOnSuccessListener(new OnSuccessListener<UploadTask.TaskSnapshot>() {
            @Override
            public void onSuccess(UploadTask.TaskSnapshot taskSnapshot) {
                //taskSnapshot.getMetadata() 는 파일의 메타데이터를 포함함(사이즈 등등)
                //DoSomething if success
                Toast.makeText(HomeActivity.this,"파일 업로드 성공",Toast.LENGTH_SHORT).show();

                ImageObject object=new ImageObject(); //하나의 파일 정보가 될것(직접 정의)
                object.imageUri=taskSnapshot.getDownloadUrl().toString(); //파일경로
                object.title=title.getText().toString(); //타이틀
                object.contents=cont.getText().toString(); //콘텐츠
                object.uid=auth.getCurrentUser().getUid(); //uid
                object.userId=auth.getCurrentUser().getEmail(); //email

                database.getReference().child("images").child(object.title).setValue(object); 
                //데이터베이스 참조 후, images 하위트리로 가서 값을 저장
            }
        });
```  
  
### ※Read Database
저장되있는 데이터를 가져올 수 잇다. (JSON 형태로 전달된다.)  
<a href=https://firebase.google.com/docs/database/android/read-and-write>참조 문서</a>
```
public class BoardActivity extends AppCompatActivity {
    private RecyclerView recyclerView;
    private List<ImageObject> list=new ArrayList<>();
    private FirebaseDatabase database;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_board);
        database=FirebaseDatabase.getInstance();

        recyclerView=findViewById(R.id.recycler);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        final BoardRecyclerViewAdapter boardRecyclerViewAdapter=new BoardRecyclerViewAdapter();
        recyclerView.setAdapter(boardRecyclerViewAdapter);

        database.getReference().child("images").addValueEventListener(new ValueEventListener() { //현재 위치는 image의 하위트리
            @Override
            public void onDataChange(DataSnapshot dataSnapshot) {
                list.clear();
                for (DataSnapshot data: dataSnapshot.getChildren()) { //2개가 저장되있으면 2번 반복
                    ImageObject object = data.getValue(ImageObject.class); //JSON -> Object
                    list.add(object);
               }
                boardRecyclerViewAdapter.notifyDataSetChanged();
            }

            @Override
            public void onCancelled(DatabaseError databaseError) {
                //DoSomething if cancelled
            }
        });

    }
    //리사이클러뷰는 설명을 생략한다.
    class BoardRecyclerViewAdapter extends RecyclerView.Adapter<BoardRecyclerViewAdapter.ItemHolder>{
        @NonNull
        @Override
        public ItemHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
            View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_board,parent,false);
            return new ItemHolder(view);
        }

        @Override
        public void onBindViewHolder(@NonNull ItemHolder holder, int position) {
            holder.textView1.setText(list.get(position).title);
            holder.textView2.setText(list.get(position).contents);

            //해당 아이템의 뷰의 이미지뷰안에 해당 주소의 이미지를 로드한다.
            Glide.with(holder.itemView.getContext()).load(list.get(position).imageUri).into(holder.imageView);
        }

        @Override
        public int getItemCount() {
            return list.size();
        }

        public class ItemHolder extends RecyclerView.ViewHolder {
            ImageView imageView;
            TextView textView1,textView2;
            public ItemHolder(View view) {
                super(view);
                imageView=view.findViewById(R.id.myimage);
                textView1=view.findViewById(R.id.info1);
                textView2=view.findViewById(R.id.info2);
            }
        }
    }
}
```
