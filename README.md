# howToUseFirebase_Storage&DB-on-Android
### ※파일 업로드  
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

                database.getReference().child("images").setValue(object); //데이터베이스 참조 후, images 하위트리로 가서 값을 저장
            }
        });
```
